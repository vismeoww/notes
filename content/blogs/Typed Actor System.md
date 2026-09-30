## Introduction 
An actor model in concurrent programming is a conceptual model for handling concurrent computation. The core concepts of an Actor model are as follows
- Processes messages asynchronously
- Has its state, which is ***not shared***
- Can do one of the three things when it receives a message
	- Send a message to other actors
	- Create new actors
	- Modify internal state and behaviour 
## Creating the project
we can start out the Rust project with the following command.

```bash
cargo new --lib typed_actor
```

but this only creates a project with a library file. we need some way to run it as well. so let's add a binary file to it. Let this be inside the folder `app`

```bash
mkdir app
touch app/main.rs
```

now change the `Cargo.toml` accordingly so that we can expose the library as well as run an example using the library functions with `cargo` command itself.

```toml
[lib]
name = "typed_actor"
path = "src/lib.rs"

[[bin]]
name = "example"
path = "app/main.rs"
```

now to run the code inside `app/main.rs` we only need to run the command 
```bash
cargo run example
```

## Using Multi Producer Single Consumer Channel

consider the following code for creating a `mpsc::channel`

```rust
use tokio::sync::{mpsc, oneshot};

#[tokio::main]
async fn main() {
    // mpsc::channel retursn a Sender (tx) and a mutable Receiver (mut rx)
    let (tx1, mut rx) = mpsc::channel(32);

    // we clone the Sender to simulate the idea of sending data from two
    // different sources, both of these senders will be pointing to the
    // same receiver
    let tx2 = tx1.clone();

    // spawns an async tokio task that sends the data to the receiver.
    // this is an async operation as the `tx1.send()` returns a future
    // on which we `await` to see if the value was sent correctly, we may
    // have to wait here if the receiver queue is full. the `move` keywords
    // here means that the value `tx1` is moved into this context and is
    // dropped as soon as the block finishes executing. ie, when the data
    // is sent
    tokio::spawn(async move {
        if let Err(_) = tx1.send(3).await {
            println!("the receiver dropped");
        }
    });

    tokio::spawn(async move {
        if let Err(_) = tx2.send(4).await {
            println!("the receiver dropped");
        }
    });

    // the Receiver will return a `Some(value)` as long as there's a Sender
    // active. and in this case when both the Senders are dropped the
    // `rx.recv().await` returns None. otherwise it polls the next message
    // from the queue and we can process the message
    while let Some(msg) = rx.recv().await {
        println!("Got {:?}", msg);
    }
}

```

## Basic Actor framework using `mpsc::Channel`
let's use the previous ideas to write two structs for `Actor` which is responsible for handling messages sent to it and an `ActorHandle` which can be used to send messages to the `Actor` . This code is inspired by [Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/)

```rust
use std::marker::Send;
use tokio::sync::mpsc;

/// The Actor struct, responsible for spawning the actor that receive the
/// messages and then handle them, the actor itself may have a state that can
/// be affected by the message
pub struct Actor<M: Send + 'static, S: Default + Send> {
    // a handle to the receiver from mpsc::channel so that we can use it to
    // receive messages
    receiver: mpsc::Receiver<M>,
    // the state of the actor that can be modified by the handle function
    state: S,
    // the state is borrowed mutably so that the function may modify it
    handle: fn(&mut S, msg: M) -> (),
}

impl<M: Send + 'static, S: Default + Send> Actor<M, S> {
    pub fn new(tx: mpsc::Receiver<M>, f: fn(&mut S, M) -> ()) -> Self {
        Actor {
            receiver: tx,
            state: S::default(),
            handle: f,
        }
    }
    pub async fn start(mut self) {
        while let Some(msg) = self.receiver.recv().await {
            (self.handle)(&mut self.state, msg)
        }
    }
}

/// ActorHandle can be used to send messages to the respective actor.
#[derive(Clone)]
pub struct ActorHandle<M: Send + 'static> {
    // holds a handle to the sender from mpsc::channel
    id: mpsc::Sender<M>,
}

impl<M: Send> ActorHandle<M> {
    // when we create a new actor what we only return is the handle, during
    // it's creation we launch the actor and create a handle to it as well.
    pub fn new<S: Default + Send + 'static>(size: usize, f: fn(&mut S, M) -> ()) -> Self {
        let (tx, rx): (mpsc::Sender<M>, mpsc::Receiver<M>) = mpsc::channel(size);
        let actor: Actor<M, S> = Actor::new(rx, f);
        tokio::spawn(async move {
            let _ = actor.start().await;
        });
        let handle = ActorHandle { id: tx };
        return handle;
    }
    pub async fn send(self, msg: M) -> () {
        if let Err(e) = self.id.send(msg).await {
            eprintln!("{:?}", e);
        }
    }
}

```
### Introducing traits
let's capture the idea of an `Actor` through a trait to make this process more straightforward
```rust
pub trait ActorTrait {
    type State : Default + Send + 'static;
    type Message : Send + 'static;
    fn handle(state: &mut Self::State, msg: Self::Message) -> ();
}
```
changing the `ActorHandle` implementation into 
```rust
    pub fn new<A : ActorTrait<Message = M>>(size: usize) -> Self {
        let (tx, rx): (mpsc::Sender<M>, mpsc::Receiver<M>) 
            = mpsc::channel(size);
        let actor: Actor<M, <A as ActorTrait>::State> 
            = Actor::new(rx, <A as ActorTrait>::handle);
        tokio::spawn(async move {
            let _ = actor.start().await;
        });
        let handle = ActorHandle { id: tx };
        return handle;
    }

```

### Extending the trait to formalise a few more actor functions

So we've set up a basic actor framework that's capable of receiving a message and calling the `handle` function on it, which processes the message and updates its own `state`
But what if we want to ask the actor something? What if we need to add a `PoisonPill` to it and terminate the actor with a cleanup code?
Let's consider the following example of `ActorTrait`

```rust
pub trait ActorTrait: 'static {
    // the state of the actor
    type State:  Send + 'static;
    // startup code to run before the actor is created, helps create the starting
    // state of the actor
    fn startup() -> Self::State;

    // messages that can be send to an actor but won't receive any reply from
    // the actor
    type Message: Send + 'static;
    // how to process such messages, this could modify the state of the actor
    fn handle(state: &mut Self::State, msg: Self::Message) -> ();

    // questions that can be asked to the actor
    type Ask: Send + 'static;
    // expected responses from the actor captured into a type
    type Answer: Send + 'static;
    // how the actor handles the questions based on the current state of the
    // actor, remember this can also modify the state
    fn ask(state: &mut Self::State, msg: Self::Ask) -> Self::Answer;

    // send a kill signal to the actor, causing it to run the cleanup code and
    // drop all the receiver handles it has
    type PoisonPill: Default + Send + 'static;
    // cleanup code to run once the actor is ready to terminate
    fn cleanup(state: &mut Self::State, signal: Self::PoisonPill) -> ();
}
```
A quick run-through of what each of these functionalities does in depth
- ***Startup*** - When an actor is created, it may want to configure itself by creating and setting certain variables within its state. Note that we don't need `State: Default` anymore because we're using `startup()-> State` to produce the state of the actor during startup
- ***Message*** - these are messages that the actor can process and do stuff with its own internal state. Note that these messages are fire and forget; whoever sends this does not expect any reply from the actor. 
- ***Queries*** - this allows an external entity to make queries to the actor, to which the actor will produce an output. This query can involve changing the state of the actor, and the query's answer itself may or may not depend on the state of the actor. The query can be made with or without a timeout
- ***Poison Pill***  - When it's time for the actor to terminate, we need to send it a signal to ask it to start termination. Upon this message being received, the actor should run the clean-up code ( which could involve sending `PoisonPill` to other actors created by this actor and other handles that need to be closed, etc.)
### Changes to `Actor` and `ActorHandle` implementation
Now let's see how this will change the implementation of both `Actor` and `ActorHandle`
```rust
pub struct Actor<A: ActorTrait> {
    // a handle to the receiver from mpsc::channel so that we can use it to
    // receive messages
    receiver: mpsc::Receiver<<A as ActorTrait>::Message>,
    // a receiver to handle poison pill
    poison_pill: mpsc::Receiver<<A as ActorTrait>::PoisonPill>,
    // a receiver to handle queries that will have a reply
    query: mpsc::Receiver<Query<A>>,
    // the state of the actor that can be modified by the handle function
    state: <A as ActorTrait>::State,
}

// the query struct contains of two things, one the query itself. And it contains
// a oneshot::Sender channel through which the reply can be sent back to the 
// place where the question came from
pub struct Query<A: ActorTrait> {
    query: <A as ActorTrait>::Ask,
    answer_channel: oneshot::Sender<<A as ActorTrait>::Answer>,
}
```

let's take a look at the function implementations for `Actor`
```rust
impl<A: ActorTrait> Actor<A> {
    pub fn new(
        rx: mpsc::Receiver<<A as ActorTrait>::Message>,
        krx: mpsc::Receiver<<A as ActorTrait>::PoisonPill>,
        arx: mpsc::Receiver<Query<A>>,
    ) -> Self {
        Actor {
            receiver: rx,
            poison_pill: krx,
            query: arx,
            // using the startup function we can define the starting state of 
            // the actor
            state: <A as ActorTrait>::startup(),
        }
    }
    pub async fn start(mut self) -> u8 {
	    // the actor runs in a forever loop awaiting values on all of the 
	    // channels on which it's listening to.
        loop {
            tokio::select! {
	            // handles the messages that are fire and forget.
	            // ie, the actor doesn't reply with anything
                Some(msg) = self.receiver.recv() => {
                    <A as ActorTrait>::handle(&mut self.state, msg);
                }
                // handles the termination of the actor, when it's terminated
                // we return `1` to indicate that this is due to a poisonPill
                Some(p) = self.poison_pill.recv() => {
	                // perform cleanup code before terminating
                    <A as ActorTrait>::cleanup(&mut self.state,p);
                    return 1;
                }
                // handles a query that's send to the actor, the query itself 
                // will contain the sender's address, once we have the answer
                // we can reply back through that channel
                Some(q) = self.query.recv() => {
                    let a = <A as ActorTrait>::ask(&mut self.state,q.query);
                    let _ = q.answer_channel.send(a);
                }
                // this code is tirggered when the last reference to the actor
                // handle is dropped
                else => {
                    eprintln!("all senders dropped");
                    <A as ActorTrait>::cleanup(
	                    &mut self.state, 
	                    <A as ActorTrait>::PoisonPill::default()
	                );
                    break;
                }
            }
        }
        return 0;
    }
}
```

and the code for `ActorHandle` as well.

```rust
#[derive(Clone)]
pub struct ActorHandle<A: ActorTrait> {
    // holds a handle to the sender from mpsc::channel
    id: mpsc::Sender<<A as ActorTrait>::Message>,
    // holds a handle to send poison pill
    kid: mpsc::Sender<<A as ActorTrait>::PoisonPill>,
    // holds a handle to send queries
    qid: mpsc::Sender<Query<A>>,
}

impl<A: ActorTrait> ActorHandle<A> {
    // when we create a new actor what we only return is the handle, during
    // it's creation we launch the actor and create a handle to it as well.
    pub fn new(size: usize) -> Self {
	    // create the channel for normal messages
        let (tx, rx): (
            mpsc::Sender<<A as ActorTrait>::Message>,
            mpsc::Receiver<<A as ActorTrait>::Message>,
        ) = mpsc::channel(size);
        // create the channel for poisonpill
        let (ktx, krx): (
            mpsc::Sender<<A as ActorTrait>::PoisonPill>,
            mpsc::Receiver<<A as ActorTrait>::PoisonPill>,
        ) = mpsc::channel(1);
        // create the channel for queries
        let (atx, arx): (
	        mpsc::Sender<Query<A>>, 
	        mpsc::Receiver<Query<A>>
	    ) = mpsc::channel(size);

        let actor: Actor<A> = Actor::new(rx, krx, arx);
        tokio::spawn(async move {
            let res = actor.start().await;
            eprintln!("Actor exited with : {}", res);
        });
        let handle = ActorHandle {
            id: tx,
            kid: ktx,
            qid: atx,
        };
        return handle;
    }
    pub async fn send(&self, msg: <A as ActorTrait>::Message) -> () {
        if let Err(e) = self.id.send(msg).await {
            eprintln!("{:?}", e);
        }
    }
    pub async fn terminate(&self, msg: <A as ActorTrait>::PoisonPill) -> () {
        if let Err(e) = self.kid.send(msg).await {
            eprintln!("{:?}", e);
        }
    }
    pub async fn ask(
        &self,
        question: <A as ActorTrait>::Ask,
    ) -> Result<<A as ActorTrait>::Answer, QueryError<A>> {
	    // first we need to create a oneshot channel through which the Actor can
	    // reply
        let (tx, rx): (
            oneshot::Sender<<A as ActorTrait>::Answer>,
            oneshot::Receiver<<A as ActorTrait>::Answer>,
        ) = oneshot::channel();
        // while making the query that's sent to the actor we attach the Sender
        // handle to it so that the actor can handle the query and reply via
        // that handle
        let q: Query<A> = Query {
            query: question,
            answer_channel: tx,
        };
        // send the query to the actor
        let _ = self.qid.send(q).await.map_err(QueryError::AskError)?;
        // wait for the answer
        // NOTE : this can be done with timeout as well
        rx.await.map_err(QueryError::AnswerError)
    }

}

#[derive(Debug)]
pub enum QueryError<A: ActorTrait> {
    AskError(SendError<Query<A>>),
    AnswerError(oneshot::error::RecvError),
}

```
let's see an example using the library that we've built
```rust
#[derive(Clone,Debug)]
pub struct TestActor {}

impl ActorTrait for TestActor {
	// the state of the actor is a tuple representing
	// (total_number_of_msg,total_sum)
    type State = (usize,i64);
    fn startup() -> Self::State {
        (0,0)
    }

    // the messages are just integers that will be added to the sum
    type Message = i64;
    // increment the total number of messages received by one 
    // add the message to the total and display current status
    fn handle(state: &mut Self::State, msg: Self::Message) -> () {
        (*state).0 +=1;
        (*state).1 += msg;
        println!("total received : {} messages, current sum : {}", 
	        (*state).0, (*state).1
	    );
    }

	// the query doesn't involve anything but asking the actor for it's 
	// current sum which is an integer
    type Ask = ();
    type Answer = i64;
    // from the state of the actor return the sum
    fn ask(state: &mut Self::State, _msg: Self::Ask) -> Self::Answer {
        (*state).1
    }

	// not really needed but upon terminating just set the state
	// back to (0,0)
    type PoisonPill = ();
    fn cleanup(state: &mut Self::State, _signal: ()) -> () {
        *state = (0,0);
    }

}

#[tokio::main]
async fn main() {
	// now that we've implemented the trait `Actor` for `TestActor` we 
	// can use that to create a handle for our actor
    let handle : Arc<ActorHandle<TestActor>> = Arc::new(ActorHandle::new(32));
    // we need to clone this handle as when we launch the async code the handle
    // is moved into the codeblock, hence creating the handle with an
    // Arc reference which is thread compatible
    let termination_handle = handle.clone();
    let ask_handle = handle.clone();

	// time until which we should wait before terminating the actor
    let terminate_time = std::time::Duration::from_millis(10000);
    // time until we should wait until asking the actor for it's current sum
    let ask_time = std::time::Duration::from_millis(7000);
    // step time to wait until we send the next message to actor
    let step = std::time::Duration::from_millis(400);

    // spawn in a loop that'll wait for the step time and then send a message 
    // to the actor. save this handle so that at the end of the `main` function 
    // we can wait on this to ensure we wait til all the messages are 
    // sent to the actor before terminating the main thread itself
    let a = tokio::spawn(async move {
        for i in 0..30 {
            handle.send(i).await;
            tokio::time::sleep(step).await;
        }
    });

    // spwan in another thread and wait until the termination time before 
    // sending the poison_pill
    tokio::spawn(async move {
        tokio::time::sleep(terminate_time).await;
        termination_handle.terminate(()).await;
    });

	// after waiting until the ask time we can ask the actor what it's current
	// state is, by then the messeg sender would've sent a bunch of integers
	// that would've accumulated in the state of the actor
    tokio::spawn(async move {
        tokio::time::sleep(ask_time).await;
        let a = ask_handle.ask(()).await;
        println!("got answer from actor {:?}",a);
    });

	// await on the first sender so that the main thread won't exit before all
	// the messages are sent. we have to awai it here and not where we create it 
	// because once we start awaiting on it the execution of the rest of
	// the `main` function is suspended. so we wait to spawn every other thread
	// that would send the messages needed before waiting on the message 
	// sender to finish sending
    let _ = a.await;
    // exit main
    ()
}
```

From the example above, we can notice that the actor will be terminated before the sender can send all the messages. This can be observed while running this code, as it is produced once the termination is complete.

```bash
Actor exited with : 1
SendError { .. }
SendError { .. }
SendError { .. }
SendError { .. }
SendError { .. }
```

before the actor is terminated we can see it processing messages and answering the query as well 

```bash
...
total received : 14 messages, current sum : 91
total received : 15 messages, current sum : 105
total received : 16 messages, current sum : 120
total received : 17 messages, current sum : 136
total received : 18 messages, current sum : 153
got answer from actor Ok(153)
total received : 19 messages, current sum : 171
total received : 20 messages, current sum : 190
total received : 21 messages, current sum : 210
...
```

changing the wait times we can see how the behaviour of the actor changes accordingly

## Limitations
as you can see from the code, the functions provided the the trait are all synchronous functions. That is when the actor handles a message or a query or when it creates itself we cannot call any other asynchronous functions inside it, this prevents the actor from creating any other actor from within itself. this can be bypassed by making the functions `async`
for example : 

```rust
async fn startup() -> Self::State ;
```

then you will notice that the compiler throws an error like this 

 ```
warning: use of `async fn` in public traits is discouraged as auto trait bounds cannot be specified
```

since we are enabling the functions from this trait as public we can solve this error by doing the following.

```rust
fn startup() -> impl Future<Output =  Self::State> + Send;
```

this still allows us to make these functions `async` and public. without having any compiler warnings.
## Thanks
the complete implementation of this project is available on my [github](https://github.com/isqnwtn/typed_actor)
checkout more of my works at :  🏠 [home](https://isqnwtn.github.io/)
