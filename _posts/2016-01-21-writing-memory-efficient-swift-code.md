---
layout: post
title: Writing memory efficient Swift code.
date: 2016-01-21
categories:
---


If you are a newcomer to Swift (or programming in general), chances are that your are overwhelmed by all those amazing posts exploring “advanced” Swift OOP techniques...

Between Monads, Lenses and Protocol Oriented Programming, let’s take a moment to talk about more fundamental stuff like memory management and especially how to avoid memory leaks.

<br>
## Introduction
###Structs vs Classes

In Swift, the entities that you create fall into two categories :

* Values Types (structs and enums)
* Reference Types (classes)

Values types are copied on assignment.

~~~swift
let array = ["Hello", "World"]
var copy = array

copy.append("!")

array // ["Hello", "World"]
copy // ["Hello", "World", "!"]
~~~

The cool thing about them is that they always have a single owner. So it’s fairly easy for swift to free up memory when exiting the scope where there are used, outside of which they are no longer needed.

Reference types are, you guessed it, passed by reference. In other words every time you assign a class instance to a variable or constant, you are creating a pointer to that instance in memory.

~~~swift
class City {
	var population = 10
}

let city = City()
let thatCity = city

thatCity.population = 100

city.population // 100
thatCity.population // 100
~~~

Since multiple parts of your code could have references to a same instance, let’s see how does the compiler(swift) knows when to free up the memory allocated when it’s no longer needed.

<br>
###ARC to the rescue !
**Automatic Reference Counting** is the compiler mechanism used by Swift (and Objective-C) to manage an app memory usages. 

Every time you create a class instance, ARC allocate a chunk memory to store informations about it, along with the values of its property. 
Whenever you assign a class instance to a variable or constant, that same variable or constant create a **strong reference** to the instance, which is basically like saying to ARC: 

> “Hey, don’t release that instance as long as i’m alive… might need to access it”.

When there are no strong reference left to that instance, ARC is allowed to dispose of that instance and reuse the memory space for something else.

Let’s illustrate all that with a good ol' exemple :

~~~swift
class Bubble {
	init () { print("Bubble created.") }
	deinit { print("Bubble poped.") }
}

var bubble: Bubble? = Bubble() // bubble holds a strong ref. on the instance
var other: Bubble? = bubble // other holds one more strong ref. on the instance

bubble = nil // other still holds a strong ref.
other = nil // no strong ref. on the instance, ARC can dispose of it
~~~

ARC is a very handful compiler feature that makes memory management seamless in most scenarios. However, it sometimes require extras infos from the developer to fully understand when does a class instance is to be deallocated…


<br>
### Automatic ? Erh … almost !
Say that you’re creating a space exploration game. Before setting out to deep space and find a tiny planet to build a colony on, you’ll need definitely need a spaceship and a pilot.

~~~swift
class Spaceship {
	var pilot: Pilot?
	
	init() { print("Spaceship created.") }
	deinit { print("Spaceship destroyed.") }
}

class Pilot {
	var spaceship: Spaceship?
	
	init() { print("Pilot created.") }
	deinit { print("Pilot destroyed.") }
}


var spaceship: Spaceship? = Spaceship() // Spaceship created.
var pilot: Pilot? = Pilot() // Pilot created.

spaceship?.pilot = pilot
pilot?.spaceship = spaceship

spaceship = nil // ...
pilot = nil // ...


~~~
See what’s happening here?

Our `Spaceship` instance hold a strong reference to the `Pilot` instance and the `Pilot` instance hold a strong reference to our `Spaceship` instance. Both instances are keeping each other alive and ARC will never deallocate them…

*BOOM !* 
Right there! An ugly **memory leak** hiding in plain sight !
<center>![Eww](/assets/eww.gif)</center>

In more ARC-ish terms, we would call this a **reference cycle**.

Hopefully, Swift defines keywords that can be used to give ARC more details about the references that we are creating :

- **strong** (default)
- **weak**
- **unowned**


Let’s walk through each reference cycle scenarios and show an example of how to resolve it using the appropriate reference type. 

## How to break retain cycles

>To help you follow along, a playground with the example classes is [available Github](https://github.com/ndethore/swift-memory-management).

### Scenario #1 - Both properties are allowed to be nil

If we consider the relationship between the Spaceship and the Pilot we created above:

- A pilot doesn’t necessarily have a spaceship. The spaceship it refers to can be deallocated without issues.
- A spaceship doesn’t necessarily have a pilot. The pilot it refers to can be deallocated without issue.

This reference cycle scenario is best solved by using **weak** references.
 
> A weak reference does not keep a strong hold on the instance it refers to and so _doesn’t no keep ARC for deallocating the referenced instance._
 
This is exactly what we need ! Beside cruising to nowhere, the spaceship should be just fine if its pilot is suddenly deallocated.

Here’s a “reference cycle free” version of the previous code snippet :

~~~swift
class Spaceship {
	weak var pilot: Pilot?
	
	init() { print("Spaceship created.") }
	deinit { print("Spaceship destroyed.") }
}

class Pilot {
	weak var spaceship: Spaceship?
	
	init() { print("Pilot created.") }
	deinit { print("Pilot destroyed.") }
}


var spaceship: Spaceship? = Spaceship() // Spaceship created.
var pilot: Pilot? = Pilot() // Pilot created.

spaceship?.pilot = pilot
pilot?.spaceship = spaceship

spaceship = nil // Pilot destroyed.
pilot = nil // Spaceship destroyed.
~~~

*Note*:  **weak** references are declared as **var** to indicate that their value can change at runtime. You’ll need to unwrap the property before accessing it’s value.

###Scenario #2 - One property is allowed to be nil and the other isn’t.

Now, say the Intergalactic Empire Consulate require every Pilot to have a License Implent in order to maneuver a ship.
Let’s create our `Implent` class and update our `Pilot` class:

~~~swift
class Implent {
	let pilot: Pilot
	
	init(pilot: Pilot) {
		self.pilot = pilot
		print("Implent created.");
	}
	
	deinit { print("Implent destroyed.") }
}

class Pilot {
	var implent: Implent?
	weak var spaceship: Spaceship?
	
	init() { print("Pilot created.") }
	deinit { print("Pilot destroyed.") }
}

var pilot: Pilot? = Pilot() // Pilot created.
pilot!.implent = Implent(pilot: pilot!) // Implent created.

pilot = nil //...

~~~

As we did for the first scenario, let’s take a moment to analyse the relationship between a Pilot and a License :

- A pilot does not necessarily have a license. His license can be deallocated without issues.
- An implent, however, is issued for a specific pilot. Therefore the pilot property of a `Implent` instance must always have a value. 

The reference cycle created in this situation is best solved using an **unowned** reference.

> Like a weak reference, an unowned reference does not keep a strong hold it refers to and doesn’t create a retain cycle. 
>
> However is it is assumed to always have a value. If the instance to which an unowned reference refers to is deallocated, accessing it will trigger a runtime error and crash.
>
> Only use when you are sure that the value exists.

Here’s an updated version of `Implent`:

~~~swift
class Implent {
	unowned let pilot: Pilot
	
	init(pilot: Pilot) {
		self.pilot = pilot
		print("Implent created.");
	}
	
	deinit { print("Implent destroyed.") }
}
~~~

*Note* : Make sure that the implent is owned by a single pilot so when it is deallocated, the implent has no more strong references to it and gets deallocated too.


###Scenario #3 - Both properties should always have a value.

Now, the last step of our spaceship assembly line is to generate an Intergalactic Registration Beacon an seal it on the main engine. This beacon is required by every spaceship.

Here’s how our `Beacon` would look like :

~~~swift
class Beacon {
	let spaceship: Spaceship
	
	init(spaceship: Spaceship) {
		self.spaceship = spaceship
		print("Beacon created.")
	}
	
	deinit { print("Beacon destroyed.") }
	
}
~~~


In this scenario :

- A Spaceship must always have a Beacon
- A Beacon is created for a specific spaceship. Therefore it must always have a non-nil reference to its Spaceship.

>Hold on a minute !

This beacon thing is quite similar to the `Implent` class we created ealier, isn’t it ? 
You guessed it. An **unowned** reference to the spaceship it is attached to is the appropriate reference type.

~~~swift
class Beacon {
	unowned let spaceship: Spaceship
	
	init(spaceship: Spaceship) {
		self.spaceship = spaceship
		print("Beacon created.")
	}
	
	deinit { print("Beacon destroyed.") }
	
}

class Spaceship {
	var beacon: Beacon!
	weak var pilot: Pilot?
	
	init() { 
		print("Spaceship created.") 
		self.beacon = Beacon(self)
	}
	
	deinit { print("Spaceship destroyed.") }
}

~~~


*Note*: The `Spaceship` initializer cannot pass `self` to the `Beacon` initializer until a new `Spaceship` is fully initialized. To cope with this requirement we can declare the beacon propperty as an implicity unwrapped optional. It will have a default value of nil, allow the two phase initialization were attempting and will still be accessible without the need to unwrap its value.


###Scenario #4 - Closure need to access instance properties or methods.

Depending on your background, you may be familiar the *blocks* in C and Objective-C or *lambdas* in other languages.  

Roughly, a *closure* is a self contained block of functionality that can be passed around your code.

It captures any constants or variable from the context in which they are defined. Also, **if you assign a closure to an instance’s property and access self’s methods and/or properties from the closure body, you’ll inevitably create a reference cycle** (the instance holds a strong reference to the closure and the closure holds a strong reference to the instance) … 

Say we add an docking procedure to our spaceship, and reference to one the station’s docking operator that we can notify once the procedure is complete : 

~~~swift

class StationOperator {
	init() { print("Station operator created.") }
	deinit { print("Station operator destroyed.") }
	
	func didEngageDockingProcedure(spaceship: Spaceship) { }
	func didFinishDockingProcedure(spaceship: Spaceship) { }
}

class Spaceship {
	var op: StationOperator?
	lazy var dockingProcedure: (Void)->Void = {
		self.op?.didEngageDockingProcedure(self)
		// do some tricky maneuver...
		self.op?.didFinishDockingProcedure(self)
	}
	
	init() {
		print("Spaceship created.")
	}
	
	deinit { print("Spaceship destroyed.") }
}

var spaceship: Spaceship? = Spaceship() // Spaceship created.
var op: StationOperator? = StationOperator() // Station operator created.

spaceship!.op = op
spaceship!.dockingProcedure()

spaceship = nil //...
op = nil //...

~~~

Preventing a reference cycle between a closure and an instance is slightly different from breaking it between two instance. 

We’ll need to declare a **capture list** at the very beginning of your closure body, before its parameter and return type (if any). Each item in capture list has either the weak or unowned keyword followed by a reference to a class instance or a variable initialized with some value.

As we did before, use the weak keyword when the captured reference may become nil at some point (This docking operator? Hm. We don’t really need him to be there the whole time.)
Use the unowned reference when the closure and the instance it refers to will always be deallocated at the same time :

~~~swift
class StationOperator {
	init() { print("Station operator created.") }
	deinit { print("Station operator destroyed.") }
	
	func didEngageDockingProcedure(spaceship: Spaceship) { }
	func didFinishDockingProcedure(spaceship: Spaceship) { }
}

class Spaceship {
	var op: StationOperator?
	lazy var dockingProcedure: (Void)->Void = {
		[unowned self, weak weakOp = self.op!] in
		
		weakOp?.didEngageDockingProcedure(self)
		// do some tricky maneuver...
		weakOp?.didFinishDockingProcedure(self)
	}
	
	init() {
		print("Spaceship created.")
	}
	
	deinit { print("Spaceship destroyed.") }
}

var spaceship: Spaceship? = Spaceship() // Spaceship created.
var op: StationOperator? = StationOperator() // Station operator created.

spaceship!.op = op
spaceship!.dockingProcedure()

spaceship = nil // Spaceship destroyed.
op = nil // Station operator destroyed.

~~~

That’s it. You’re ready to write leak free swift code and keep your app’s memory footprint as low as possible.
<center>![Rockstar](/assets/badass.gif)</center>
---
#Summary


* Be aware of strong reference cycles when you need two class instances to have references to each other.
* Use weak references when the instance referred is allowed to become nil.
* Use unowned references when the instance referred cannot become nil.
* User capture list when you assign a closure to a property of an instance and that closure captures the instance.


**Happy coding !** 

---
*[Edited on 2016-01-22]:* 

A playground with the code samples above is [available on Github](https://github.com/ndethore/swift-memory-management).
Pull requests are very welcomed 🤗







