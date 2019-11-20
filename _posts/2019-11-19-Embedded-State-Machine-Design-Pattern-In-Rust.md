---
layout: post
title: Embedded state-machine design pattern in Rust
---

Lately at S2E we have been thinking a lot about [finite state machines]([https://en.wikipedia.org/wiki/Finite-state_machine) in embedded systems and which design pattern is best suited for implementing them in our systems. We mostly use C for our embedded developments but we found this was a very good opportunity to try to implement a state-machine design pattern in Rust and explore a bit of the languages possibilities. This blog post is an attempt to document our findings and lessons during the process.

**Summary:** In the end this became quite a long post. The short version of it is that we implement states by creating objects that implement a State trait and transitions by using closures without input argument that capture the needed inputs from their scope. The state machine is basically an array of references to state objects and a 2-dimensional array of references to function objects. Here is the [playground link to the final solution](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f8903b43c10b733b7d325c86a60e64f1). Don't know how such a state machine implementation would extend in real-life but this is was a very good Rust exploration exercise.

## State-machine definition and design goals

There are two particular aspect of state-machines that are often seen in embedded systems in comparison to the ones found, for example, in GUI implementations. One aspect is that the state-machine is evaluated on the tick of a clock and that a given action is executed inside each state, even when there is no external or transition input (other than the clock tick). Another difference is that the transitions can happen not only due to external events (e.g. a button press) but also from conditions within the states themselves (e.g. a counter reaching a value). 

For this example we will be using a very simple state machine with two states A and B that can transition between each other. Each state has three functions associated to it: on_entry (to be executed when first entering the state), during (to be executed on clock tick) and on_exit (to be executed when leaving the state). In our example each state has an associated counter, which get increased on every clock time. The transition from state A to state B happens when  the internal count_a reaches 5. The transition from state B to state A takes place when an external variable *x* has a value of *true*. Figure 1 shows the UML state-diagram of our simple example state machine.

![Simple state machine diagram](C:\workcopies\s2e-systems.github.io\images\simple_state_machine_uml.png)*Figure 1: State-diagram of example state-machine*

Our design goals for this state machine were:

1. Have the state logic and the transition logic completely separate. This allows reusing the same state objects in a different state-machine with potentially different transitions.
2. Be able to "visualize" the whole state-machine in "one screen". What is meant by looking at a very small portion of code one should be able to make out which states exists and which transitions are possible, while not caring about the implementation details.
3. Coming from the *embedded C* world we want to avoid global (or static lifetime) variables. Encapsulation and pure functions are something we hold dear.
4. Have a generic execution function that can be re-used by different state-machines.
5. Since we are targeting embedded systems, ideally avoid heap allocations.
6. Ideally have named rather than numbered states.
7. And just like about any software, be easily extendable, testable, robust, ...

Looking around the Internet and software books, we found 3 major groups of implementations for finite-state machines:

- **The enum + switch/case combination**: In the "simplest" implementation one creates an enum with all the state and implement a switch/case statement that implement the functionality and transition of each state. For our very simple example it would work just fine but our experience (which seems to be shared by many others) is that this does not scale well and quickly leads to complex case statements and lots of code repetition. It breaks pretty much all our design goals.

- **The GoF (Gamma et. al, 1994) State Pattern and its many derivations:**  Using this pattern, the implementation provides a client interface to inject the event into the state-machine and respond by the adequate state changes. We felt it was a good solution for more generic state-machines without the specific particularities of the embedded state-machine we are targeting in this example.  [Here](https://hoverbear.org/blog/rust-state-machine-pattern/) is a nice blog post using Rust to create an interesting variation on this pattern. In the form presented in the Design Patterns: Elements of Reusable Object-Oriented Software book it breaks our design goals 1, 2 and 4.

- **The state array and state-transition table pattern**: In this case, the state machine is implemented using an array of states and a state-transition table (array of arrays) with pointers to functions which are used to evaluate if a given transition is valid or not. In a naïve way, this pattern breaks only our design goal 5 since states are access as an index in an array. 

  Though, when coding in C, the evaluation of state transitions can quickly become complex (or turn to global object) since the signatures of the state transition functions can be wildly varying (in our little example, one transition depends on a counter from a state and the other on a boolean from an input). But Rust is providing many other possibilities so this is the method that we decided to go for. 

Now that we have settle on a pattern, lets see how we can achieve it in Rust.

## State table pattern in Rust

### State objects

To get our state-machine going we need to be able to define state objects. Every state needs to provide the *on_entry*, *during* and *on_exit* methods so this can be easily achieved with a State trait:

```rust
trait State {
    fn on_entry(&mut self) {}
    fn during(&mut self) {}
    fn on_exit(&mut self) {}
}
```

The trait provides a default empty implementation for each of the methods since it is up to each state to provide whatever methods make sense for itself. Otherwise, there are no external dependencies in the function call

We can now implement our state objects which provide their own State implementation. For example for StateA we have:

```rust
struct StateA {
    count_a: i32,
}

impl State for StateA {
    fn during(&mut self) {
        println!("During A with value {}", self.count_a);
        self.count_a += 1;
    }

    fn on_entry(&mut self) {
        self.count_a = 0;
    }
}

impl StateA {
    fn new() -> Self {
        StateA { count_a: 0 }
    }
}
```

### State-machine class

For our example state-machine from Figure 1, the state-transition table looks like:

| State \ Target state |              A              |              B              |
| :------------------: | :-------------------------: | :-------------------------: |
|        **A**         |              -              | transition_a_to_b() -> bool |
|        **B**         | transition_b_to_a() -> bool |              -              |

Whenever the transition function returns true, that means that the transition should be executed. This means that all transition should be evaluated based on a function which receives no input and returns a bool.

Now that we have both states and a generic signature for our transition functions, we can implement the actual state-machine object and "executor":

```rust
struct StateMachine<'a> {
    states: &'a mut [&'a mut dyn State],
    transitions: &'a[ &'a [&'a dyn Fn() -> bool ] ],
    state_index: usize,
}

impl StateMachine<'_> {
    fn execute(&mut self) {
        self.states[self.state_index].during();
        for (index, transition) in self.transitions[self.state_index].iter().enumerate() {
            if transition() == true {
                self.states[self.state_index].on_exit();
                self.states[index].on_entry();
                self.state_index = index;
            }
        }
    }
}
```

The state-machine object is composed by:

- A pointer to an array (in fact a slice) of pointers to objects which implement the State trait `states: &'a mut [&'a mut dyn State],`
- A pointer to a 2-dimensional array of pointers to function objects with no arguments and returning a bool `transitions: &'a[ &'a [&'a dyn Fn() -> bool ] ],`
- An index pointing to the actual state of the state machine `state_index: usize,`

The state-machine execution calls the *during* function of the current state and then iterates through the corresponding index of the transition function for that state to check if a valid transition exists, in which case the *on_exit* and *on_entry* functions of the previous and new state are called. We have implemented all the transitions to be executed on the same execution step but this could easily be changed to a different order, for example, the entry function being executed the step after the exit function.

A big assumption here is that the order of the rows of the transition table corresponds to order of the states in the state vector. This assumption allows the execution to depend only on an iteration over a row of the transition table, which is linearly dependent on the number of state (O(n)). For additional decoupling, the state number could also be stored and additional searches for the transition and/or for the state execution could be performed. This would make the running time to O(n^2) or O(n^3). For our purposes we decided to go down the "simpler" route and use an index based implementation.

### State-machine usage

Now that we have our state objects and state machine execution algorithm we need to define the transition functions. The main problem against using the state-transition table pattern in C is that different transitions need different inputs, thus resulting in different function signatures. To group these functions in an array of function pointers in a sane way requires using global objects and keeping the function signature as `bool (*transition)(void)`.  In Rust we can get around this limitation using closures, which generate a function signature of `Fn() -> bool` while capturing its environment as required. Here is a snippet of how the implementation looks like:

```rust
fn no_transition() -> bool {
    false
}

fn main() {
	let mut x = false;
    
    let mut state_a = StateA::new();
    let mut state_b = StateB::new();
    
    let transition_a_to_b = || {
        if state_a.count_a > 4 {
            return true;
        }
        false
    };
    
    let transition_b_to_a = || {
        if x == true {
            return true;
        }
        false
    };
    
    let transitions_a : [&dyn Fn()->bool ; 2] = [&no_transition, &transition_a_to_b];
    let transitions_b : [&dyn Fn()->bool ; 2] = [&transition_b_to_a, &no_transition];
    
    let transitions = [&transitions_a[..], &transitions_b[..]];
    
    ...
}
```

The generated closures *transition_a_to_b* and *transition_b_to_a* implement the *Fn()->bool* trait and can be grouped together in a vector with the *no_transition* function. Closures that have no capture arguments automatically capture the needed variables on the scope in which they are defined, so transitions can be elegantly made dependent on either inputs or state elements.

We are ready to define the complete state-machine and execute some operations with it. The second part of our main function looks like this:

```rust
fn main() {
    ...
    let mut states : [&mut dyn State;2] = [&mut state_a, &mut state_b];
    
    let mut state_machine = StateMachine{
        states: &mut states,
        transitions: &transitions[..],
        state_index: 0};
    
    println!("Start state machine!");
    
    let mut counter = 1;
    loop{
        state_machine.execute();
        
        counter = counter + 1;
        
        if counter > 7 {
            x = true;
        }
        
        if counter > 15 {
            break;
        }
    }
    
    println!("Done");
}
```

Here is the [playground link for the complete code](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=2e9fe664d8200778d5657171159dc0ec). The solution looked good but unfortunately the compilation fails with the following complaints from the borrow checker:

```bash
error[E0502]: cannot borrow `state_a` as mutable because it is also borrowed as immutable
  --> src/main.rs:93:44
   |
74 |     let transition_a_to_b = || {
   |                             -- immutable borrow occurs here
75 |         if state_a.count_a > 4 {
   |            ------- first borrow occurs due to use of `state_a` in closure
...
93 |     let mut states : [&mut dyn State;2] = [&mut state_a, &mut state_b];
   |                                            ^^^^^^^^^^^^ mutable borrow occurs here
...
97 |         transitions: &transitions[..],
   |                       ----------- immutable borrow later used here

error: aborting due to previous error

error[E0506]: cannot assign to `x` because it is borrowed
   --> src/main.rs:109:13
    |
81  |     let transition_b_to_a = || {
    |                             -- borrow of `x` occurs here
82  |         if x == true {
    |            - borrow occurs due to use in closure
...
104 |         state_machine.execute();
    |         ------------- borrow later used here
...
109 |             x = true;
    |             ^^^^^^^^ assignment to borrowed `x` occurs here

```

Even though *state_a* is borrowed into the closure it is not used until the function is actually called so this should in theory not be a problem, but it is probably hard for the borrow checker to figure such things out in general cases. The same happens for the input variable *x*.

### Satisfying the borrow checker

To convince the borrow checker that what we are doing is correct, we have to make use of *RefCell* to check the borrowing at run-time. This is of course a bit of a compromise but we have not found a better way in this case.  Using the RefCell, we need to modify the implementation of our state-machine class such that the states become references to a RefCell:

```rust
struct StateMachine<'a> {
    states: &'a mut [&'a RefCell<dyn State>],
    transitions: &'a[ &'a [&'a dyn Fn() -> bool ] ],
    state: usize,
}

impl StateMachine<'_> {
    
    fn execute(&mut self) {
        self.states[self.state].borrow_mut().during();
        for (index, transition) in self.transitions[self.state].iter().enumerate() {
            if transition() == true {
                self.states[self.state].borrow_mut().exit();
                self.states[index].borrow_mut().init();
                self.state = index;
            }
        }
    }
}
```

On the state-machine implementation side we also use the RefCell both for the states and the input variables used in the transition. This is the complete main functions:

```rust
fn main() {
	let x = RefCell::new(false);
    
    let state_a = RefCell::new(StateA::new());
    let state_b = RefCell::new(StateB::new());
    
    let transition_a_to_b = || {
        if state_a.borrow().count_a > 4 {
            return true;
        }
        false
    };
    
    let transition_b_to_a = || {
        if *x.borrow() == true {
            return true;
        }
        false
    };
    
    let transitions_a : [&dyn Fn()->bool ; 2] = [&no_transition, &transition_a_to_b];
    let transitions_b : [&dyn Fn()->bool ; 2] = [&transition_b_to_a, &no_transition];
    
    let transitions = [&transitions_a[..], &transitions_b[..]];
    
    let mut states : [& RefCell<dyn State>;2] = [&state_a, &state_b];
    
    let mut state_machine = StateMachine{
        states: &mut states,
        transitions: &transitions[..],
        state_index: 0};
    
    println!("Start state machine!");
    
    let mut counter = 1;
    loop{
        state_machine.execute();
        
        counter = counter + 1;
        
        if counter > 7 {
            x.replace(true);
        }
        
        if counter > 15 {
            break;
        }
    }
    
    println!("Done");
}
```

Now everything compiles and runs without problems. This is the output:

```bash
Start state machine!
During A with value 0
During A with value 1
During A with value 2
During A with value 3
During A with value 4
During B with value 10
During B with value 9
During B with value 8
During A with value 0
During A with value 1
During A with value 2
During A with value 3
During A with value 4
During B with value 7
During A with value 0
Done
```

Here is the [playground link for the complete solution](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f8903b43c10b733b7d325c86a60e64f1).

## Conclusion

Using the closures traits it is possible to implement a generic state-machine pattern using the State Table pattern in what looks like a fairly nice way. The fact that the borrow checker obliged us to use RefCells makes the interface somewhat more clumsy but it still doesn't feel too bad. It though still requires some attention from the user to create the arrays in the right order and in the right dimension (whenever const generics come, size restrictions could be built into the state machine definition for some additional guarantees)

Going back to our design goals, it feels like one of the main missing points is having named rather than numbered states. The unit testing of transitions is also probably a problem since it it would be hard to move the closures out of the place they were defined in. A different way could be either to Box the objects and pass ownership to the state machine itself (probably some additional effort managing references and lifetimes) or to also create transition objects in the same style as the state objects (in effect, emulating how closures are created behind the scenes) and pay the price in additional verbosity for creating and managing transitions.

Don't know yet how such a state machine implementation would extend in real-life but this is was a very good Rust exploration exercise. Fortunately the language is flexible enough to allow us to implement different patterns in fairly elegant ways to suit different needs.