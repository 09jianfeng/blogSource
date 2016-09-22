---
title: ReactiveCocoa文档
date: 2016-09-08 07:47:03
tags: [iOS,Objective-C,MVVM]
---

参考：

<https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/Documentation>

 这里做一些对官方文档的要点记录，如果要更详细了解建议看原文, [官方文档](<https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/Documentation>)
 
 # ReactiveCocoa

_NOTE: This is legacy introduction to the Objective-C ReactiveCocoa. For the
updated version that uses Swift please see the main [README][]_

ReactiveCocoa (RAC) is an Objective-C framework inspired by [Functional Reactive
Programming][]. It provides APIs for **composing and transforming streams of
values**.

If you're already familiar with functional reactive programming or know the basic
premise of ReactiveCocoa, check out the other documentation in this folder for a
framework overview and more in-depth information about how it all works in practice.

## New to ReactiveCocoa?

ReactiveCocoa is documented like crazy, and there's a wealth of introductory
material available to explain what RAC is and how you can use it.

If you want to learn more, we recommend these resources, roughly in order:

 1. [Introduction](#introduction)
 1. [When to use ReactiveCocoa](#when-to-use-reactivecocoa)
 1. [Framework Overview][]
 1. [Basic Operators][]
 1. [Header documentation](../../ReactiveCocoa/Objective-C)
 1. Previously answered [Stack Overflow](https://github.com/ReactiveCocoa/ReactiveCocoa/wiki)
    questions and [GitHub issues](https://github.com/ReactiveCocoa/ReactiveCocoa/issues?labels=question&state=closed)
 1. The rest of this folder
 1. [Functional Reactive Programming on iOS](https://leanpub.com/iosfrp/) 
    (eBook)
 
If you have any further questions, please feel free to [file an issue](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/new). 

## Introduction

ReactiveCocoa is inspired by [functional reactive
programming](http://blog.maybeapps.com/post/42894317939/input-and-output).
Rather than using mutable variables which are replaced and modified in-place,
RAC provides signals (represented by `RACSignal`) that capture present and
future values.

By chaining, combining, and reacting to signals, software can be written
declaratively, without the need for code that continually observes and updates
values.

For example, a text field can be bound to the latest time, even as it changes,
instead of using additional code that watches the clock and updates the
text field every second.  It works much like KVO, but with blocks instead of
overriding `-observeValueForKeyPath:ofObject:change:context:`.

Signals can also represent asynchronous operations, much like [futures and
promises][]. This greatly simplifies asynchronous software, including networking
code.

One of the major advantages of RAC is that it provides a single, unified
approach to dealing with asynchronous behaviors, including delegate methods,
callback blocks, target-action mechanisms, notifications, and KVO.

Here's a simple example:

```objc
// When self.username changes, logs the new name to the console.
//
// RACObserve(self, username) creates a new RACSignal that sends the current
// value of self.username, then the new value whenever it changes.
// -subscribeNext: will execute the block whenever the signal sends a value.
[RACObserve(self, username) subscribeNext:^(NSString *newName) {
	NSLog(@"%@", newName);
}];
```

But unlike KVO notifications, signals can be chained together and operated on:

```objc
// Only logs names that starts with "j".
//
// -filter returns a new RACSignal that only sends a new value when its block
// returns YES.
[[RACObserve(self, username)
	filter:^(NSString *newName) {
		return [newName hasPrefix:@"j"];
	}]
	subscribeNext:^(NSString *newName) {
		NSLog(@"%@", newName);
	}];
```

Signals can also be used to derive state. Instead of observing properties and
setting other properties in response to the new values, RAC makes it possible to
express properties in terms of signals and operations:

```objc
// Creates a one-way binding so that self.createEnabled will be
// true whenever self.password and self.passwordConfirmation
// are equal.
//
// RAC() is a macro that makes the binding look nicer.
// 
// +combineLatest:reduce: takes an array of signals, executes the block with the
// latest value from each signal whenever any of them changes, and returns a new
// RACSignal that sends the return value of that block as values.
RAC(self, createEnabled) = [RACSignal 
	combineLatest:@[ RACObserve(self, password), RACObserve(self, passwordConfirmation) ] 
	reduce:^(NSString *password, NSString *passwordConfirm) {
		return @([passwordConfirm isEqualToString:password]);
	}];
```

Signals can be built on any stream of values over time, not just KVO. For
example, they can also represent button presses:

```objc
// Logs a message whenever the button is pressed.
//
// RACCommand creates signals to represent UI actions. Each signal can
// represent a button press, for example, and have additional work associated
// with it.
//
// -rac_command is an addition to NSButton. The button will send itself on that
// command whenever it's pressed.
self.button.rac_command = [[RACCommand alloc] initWithSignalBlock:^(id _) {
	NSLog(@"button was pressed!");
	return [RACSignal empty];
}];
```

Or asynchronous network operations:

```objc
// Hooks up a "Log in" button to log in over the network.
//
// This block will be run whenever the login command is executed, starting
// the login process.
self.loginCommand = [[RACCommand alloc] initWithSignalBlock:^(id sender) {
	// The hypothetical -logIn method returns a signal that sends a value when
	// the network request finishes.
	return [client logIn];
}];

// -executionSignals returns a signal that includes the signals returned from
// the above block, one for each time the command is executed.
[self.loginCommand.executionSignals subscribeNext:^(RACSignal *loginSignal) {
	// Log a message whenever we log in successfully.
	[loginSignal subscribeCompleted:^{
		NSLog(@"Logged in successfully!");
	}];
}];

// Executes the login command when the button is pressed.
self.loginButton.rac_command = self.loginCommand;
```

Signals can also represent timers, other UI events, or anything else that
changes over time.

Using signals for asynchronous operations makes it possible to build up more
complex behavior by chaining and transforming those signals. Work can easily be
triggered after a group of operations completes:

```objc
// Performs 2 network operations and logs a message to the console when they are
// both completed.
//
// +merge: takes an array of signals and returns a new RACSignal that passes
// through the values of all of the signals and completes when all of the
// signals complete.
//
// -subscribeCompleted: will execute the block when the signal completes.
[[RACSignal 
	merge:@[ [client fetchUserRepos], [client fetchOrgRepos] ]] 
	subscribeCompleted:^{
		NSLog(@"They're both done!");
	}];
```

Signals can be chained to sequentially execute asynchronous operations, instead
of nesting callbacks with blocks. This is similar to how [futures and promises][]
are usually used:

```objc
// Logs in the user, then loads any cached messages, then fetches the remaining
// messages from the server. After that's all done, logs a message to the
// console.
//
// The hypothetical -logInUser methods returns a signal that completes after
// logging in.
//
// -flattenMap: will execute its block whenever the signal sends a value, and
// returns a new RACSignal that merges all of the signals returned from the block
// into a single signal.
[[[[client 
	logInUser] 
	flattenMap:^(User *user) {
		// Return a signal that loads cached messages for the user.
		return [client loadCachedMessagesForUser:user];
	}]
	flattenMap:^(NSArray *messages) {
		// Return a signal that fetches any remaining messages.
		return [client fetchMessagesAfterMessage:messages.lastObject];
	}]
	subscribeNext:^(NSArray *newMessages) {
		NSLog(@"New messages: %@", newMessages);
	} completed:^{
		NSLog(@"Fetched all messages.");
	}];
```

RAC even makes it easy to bind to the result of an asynchronous operation:

```objc
// Creates a one-way binding so that self.imageView.image will be set as the user's
// avatar as soon as it's downloaded.
//
// The hypothetical -fetchUserWithUsername: method returns a signal which sends
// the user.
//
// -deliverOn: creates new signals that will do their work on other queues. In
// this example, it's used to move work to a background queue and then back to the main thread.
//
// -map: calls its block with each user that's fetched and returns a new
// RACSignal that sends values returned from the block.
RAC(self.imageView, image) = [[[[client 
	fetchUserWithUsername:@"joshaber"]
	deliverOn:[RACScheduler scheduler]]
	map:^(User *user) {
		// Download the avatar (this is done on a background queue).
		return [[NSImage alloc] initWithContentsOfURL:user.avatarURL];
	}]
	// Now the assignment will be done on the main thread.
	deliverOn:RACScheduler.mainThreadScheduler];
```

That demonstrates some of what RAC can do, but it doesn't demonstrate why RAC is
so powerful. It's hard to appreciate RAC from README-sized examples, but it
makes it possible to write code with less state, less boilerplate, better code
locality, and better expression of intent.

For more sample code, check out [C-41][] or [GroceryList][], which are real iOS
apps written using ReactiveCocoa. Additional information about RAC can be found
in this folder.

## When to use ReactiveCocoa

Upon first glance, ReactiveCocoa is very abstract, and it can be difficult to
understand how to apply it to concrete problems.

Here are some of the use cases that RAC excels at.

### Handling asynchronous or event-driven data sources

Much of Cocoa programming is focused on reacting to user events or changes in
application state. Code that deals with such events can quickly become very
complex and spaghetti-like, with lots of callbacks and state variables to handle
ordering issues.

Patterns that seem superficially different, like UI callbacks, network
responses, and KVO notifications, actually have a lot in common. [RACSignal][]
unifies all these different APIs so that they can be composed together and
manipulated in the same way.

For example, the following code:

```objc

static void *ObservationContext = &ObservationContext;

- (void)viewDidLoad {
	[super viewDidLoad];

	[LoginManager.sharedManager addObserver:self forKeyPath:@"loggingIn" options:NSKeyValueObservingOptionInitial context:&ObservationContext];
	[NSNotificationCenter.defaultCenter addObserver:self selector:@selector(loggedOut:) name:UserDidLogOutNotification object:LoginManager.sharedManager];

	[self.usernameTextField addTarget:self action:@selector(updateLogInButton) forControlEvents:UIControlEventEditingChanged];
	[self.passwordTextField addTarget:self action:@selector(updateLogInButton) forControlEvents:UIControlEventEditingChanged];
	[self.logInButton addTarget:self action:@selector(logInPressed:) forControlEvents:UIControlEventTouchUpInside];
}

- (void)dealloc {
	[LoginManager.sharedManager removeObserver:self forKeyPath:@"loggingIn" context:ObservationContext];
	[NSNotificationCenter.defaultCenter removeObserver:self];
}

- (void)updateLogInButton {
	BOOL textFieldsNonEmpty = self.usernameTextField.text.length > 0 && self.passwordTextField.text.length > 0;
	BOOL readyToLogIn = !LoginManager.sharedManager.isLoggingIn && !self.loggedIn;
	self.logInButton.enabled = textFieldsNonEmpty && readyToLogIn;
}

- (IBAction)logInPressed:(UIButton *)sender {
	[[LoginManager sharedManager]
		logInWithUsername:self.usernameTextField.text
		password:self.passwordTextField.text
		success:^{
			self.loggedIn = YES;
		} failure:^(NSError *error) {
			[self presentError:error];
		}];
}

- (void)loggedOut:(NSNotification *)notification {
	self.loggedIn = NO;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
	if (context == ObservationContext) {
		[self updateLogInButton];
	} else {
		[super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
	}
}
```

… could be expressed in RAC like so:

```objc
- (void)viewDidLoad {
	[super viewDidLoad];

	@weakify(self);

	RAC(self.logInButton, enabled) = [RACSignal
		combineLatest:@[
			self.usernameTextField.rac_textSignal,
			self.passwordTextField.rac_textSignal,
			RACObserve(LoginManager.sharedManager, loggingIn),
			RACObserve(self, loggedIn)
		] reduce:^(NSString *username, NSString *password, NSNumber *loggingIn, NSNumber *loggedIn) {
			return @(username.length > 0 && password.length > 0 && !loggingIn.boolValue && !loggedIn.boolValue);
		}];

	[[self.logInButton rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(UIButton *sender) {
		@strongify(self);

		RACSignal *loginSignal = [LoginManager.sharedManager
			logInWithUsername:self.usernameTextField.text
			password:self.passwordTextField.text];

			[loginSignal subscribeError:^(NSError *error) {
				@strongify(self);
				[self presentError:error];
			} completed:^{
				@strongify(self);
				self.loggedIn = YES;
			}];
	}];

	RAC(self, loggedIn) = [[NSNotificationCenter.defaultCenter
		rac_addObserverForName:UserDidLogOutNotification object:nil]
		mapReplace:@NO];
}
```

### Chaining dependent operations

Dependencies are most often found in network requests, where a previous request
to the server needs to complete before the next one can be constructed, and so
on:

```objc
[client logInWithSuccess:^{
	[client loadCachedMessagesWithSuccess:^(NSArray *messages) {
		[client fetchMessagesAfterMessage:messages.lastObject success:^(NSArray *nextMessages) {
			NSLog(@"Fetched all messages.");
		} failure:^(NSError *error) {
			[self presentError:error];
		}];
	} failure:^(NSError *error) {
		[self presentError:error];
	}];
} failure:^(NSError *error) {
	[self presentError:error];
}];
```

ReactiveCocoa makes this pattern particularly easy:

```objc
[[[[client logIn]
	then:^{
		return [client loadCachedMessages];
	}]
	flattenMap:^(NSArray *messages) {
		return [client fetchMessagesAfterMessage:messages.lastObject];
	}]
	subscribeError:^(NSError *error) {
		[self presentError:error];
	} completed:^{
		NSLog(@"Fetched all messages.");
	}];
```

### Parallelizing independent work

Working with independent data sets in parallel and then combining them into
a final result is non-trivial in Cocoa, and often involves a lot of
synchronization:

```objc
__block NSArray *databaseObjects;
__block NSArray *fileContents;
 
NSOperationQueue *backgroundQueue = [[NSOperationQueue alloc] init];
NSBlockOperation *databaseOperation = [NSBlockOperation blockOperationWithBlock:^{
	databaseObjects = [databaseClient fetchObjectsMatchingPredicate:predicate];
}];

NSBlockOperation *filesOperation = [NSBlockOperation blockOperationWithBlock:^{
	NSMutableArray *filesInProgress = [NSMutableArray array];
	for (NSString *path in files) {
		[filesInProgress addObject:[NSData dataWithContentsOfFile:path]];
	}

	fileContents = [filesInProgress copy];
}];
 
NSBlockOperation *finishOperation = [NSBlockOperation blockOperationWithBlock:^{
	[self finishProcessingDatabaseObjects:databaseObjects fileContents:fileContents];
	NSLog(@"Done processing");
}];
 
[finishOperation addDependency:databaseOperation];
[finishOperation addDependency:filesOperation];
[backgroundQueue addOperation:databaseOperation];
[backgroundQueue addOperation:filesOperation];
[backgroundQueue addOperation:finishOperation];
```

The above code can be cleaned up and optimized by simply composing signals:

```objc
RACSignal *databaseSignal = [[databaseClient
	fetchObjectsMatchingPredicate:predicate]
	subscribeOn:[RACScheduler scheduler]];

RACSignal *fileSignal = [RACSignal startEagerlyWithScheduler:[RACScheduler scheduler] block:^(id<RACSubscriber> subscriber) {
	NSMutableArray *filesInProgress = [NSMutableArray array];
	for (NSString *path in files) {
		[filesInProgress addObject:[NSData dataWithContentsOfFile:path]];
	}

	[subscriber sendNext:[filesInProgress copy]];
	[subscriber sendCompleted];
}];

[[RACSignal
	combineLatest:@[ databaseSignal, fileSignal ]
	reduce:^ id (NSArray *databaseObjects, NSArray *fileContents) {
		[self finishProcessingDatabaseObjects:databaseObjects fileContents:fileContents];
		return nil;
	}]
	subscribeCompleted:^{
		NSLog(@"Done processing");
	}];
```

### Simplifying collection transformations

Higher-order functions like `map`, `filter`, `fold`/`reduce` are sorely missing
from Foundation, leading to loop-focused code like this:

```objc
NSMutableArray *results = [NSMutableArray array];
for (NSString *str in strings) {
	if (str.length < 2) {
		continue;
	}

	NSString *newString = [str stringByAppendingString:@"foobar"];
	[results addObject:newString];
}
```

[RACSequence][] allows any Cocoa collection to be manipulated in a uniform and
declarative way:

```objc
RACSequence *results = [[strings.rac_sequence
	filter:^ BOOL (NSString *str) {
		return str.length >= 2;
	}]
	map:^(NSString *str) {
		return [str stringByAppendingString:@"foobar"];
	}];
```

## System Requirements

ReactiveCocoa supports OS X 10.8+ and iOS 8.0+.

## Importing ReactiveCocoa

To add RAC to your application:

 1. Add the ReactiveCocoa repository as a submodule of your application's
    repository.
 1. Run `script/bootstrap` from within the ReactiveCocoa folder.
 1. Drag and drop `ReactiveCocoa.xcodeproj` into your
    application's Xcode project or workspace.
 1. On the "Build Phases" tab of your application target, add RAC to the "Link
    Binary With Libraries" phase.
    * **On iOS**, add `libReactiveCocoa-iOS.a`.
    * **On OS X**, add `ReactiveCocoa.framework`. RAC must also be added to any
      "Copy Frameworks" build phase. If you don't already have one, simply add
      a "Copy Files" build phase and target the "Frameworks" destination.
 1. Add `"$(BUILD_ROOT)/../IntermediateBuildFilesPath/UninstalledProducts/include"
    $(inherited)` to the "Header Search Paths" build setting (this is only
    necessary for archive builds, but it has no negative effect otherwise).
 1. **For iOS targets**, add `-ObjC` to the "Other Linker Flags" build setting.
 1. **If you added RAC to a project (not a workspace)**, you will also need to
    add the appropriate RAC target to the "Target Dependencies" of your
    application.

If you would prefer to use [CocoaPods](http://cocoapods.org), there are some
[ReactiveCocoa
podspecs](https://github.com/CocoaPods/Specs/tree/master/Specs/ReactiveCocoa) that
have been generously contributed by third parties.

To see a project already set up with RAC, check out [C-41][] or [GroceryList][],
which are real iOS apps written using ReactiveCocoa.

## More Info

ReactiveCocoa is inspired by .NET's [Reactive
Extensions](http://msdn.microsoft.com/en-us/data/gg577609) (Rx). Most of the
principles of Rx apply to RAC as well. There are some really good Rx resources
out there:

* [Reactive Extensions MSDN entry](http://msdn.microsoft.com/en-us/library/hh242985.aspx)
* [Reactive Extensions for .NET Introduction](http://leecampbell.blogspot.com/2010/08/reactive-extensions-for-net.html)
* [Rx - Channel 9 videos](http://channel9.msdn.com/tags/Rx/)
* [Reactive Extensions wiki](http://rxwiki.wikidot.com/)
* [101 Rx Samples](http://rxwiki.wikidot.com/101samples)
* [Programming Reactive Extensions and LINQ](http://www.amazon.com/Programming-Reactive-Extensions-Jesse-Liberty/dp/1430237473)

RAC and Rx are both frameworks inspired by functional reactive programming. Here 
are some resources related to FRP:

* [What is FRP? - Elm Language](http://elm-lang.org/learn/What-is-FRP.elm)
* [What is Functional Reactive Programming - Stack Overflow](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming/1030631#1030631)
* [Specification for a Functional Reactive Language - Stack Overflow](http://stackoverflow.com/questions/5875929/specification-for-a-functional-reactive-programming-language#5878525)
* [Escape from Callback Hell](http://elm-lang.org/learn/Escape-from-Callback-Hell.elm)
* [Principles of Reactive Programming on Coursera](https://www.coursera.org/course/reactive)

[README]: ../../README.md
[Basic Operators]: BasicOperators.md
[Framework Overview]: FrameworkOverview.md
[Functional Reactive Programming]: http://en.wikipedia.org/wiki/Functional_reactive_programming
[GroceryList]:  https://github.com/jspahrsummers/GroceryList
[RACSequence]: ../../ReactiveCocoa/Objective-C/RACSequence.h
[RACSignal]: ../../ReactiveCocoa/Objective-C/RACSignal.h
[futures and promises]: http://en.wikipedia.org/wiki/Futures_and_promises
[C-41]: https://github.com/AshFurrow/C-41


# Basic Operators

This document explains some of the most common operators used in ReactiveCocoa,
and includes examples demonstrating their use.

Operators that apply to [sequences][Sequences] _and_ [signals][Signals] are
known as [stream][Streams] operators.

**[Performing side effects with signals](#performing-side-effects-with-signals)**

 1. [Subscription](#subscription)
 1. [Injecting effects](#injecting-effects)

**[Transforming streams](#transforming-streams)**

 1. [Mapping](#mapping)
 1. [Filtering](#filtering)

**[Combining streams](#combining-streams)**

 1. [Concatenating](#concatenating)
 1. [Flattening](#flattening)
 1. [Mapping and flattening](#mapping-and-flattening)

**[Combining signals](#combining-signals)**

 1. [Sequencing](#sequencing)
 1. [Merging](#merging)
 1. [Combining latest values](#combining-latest-values)
 1. [Switching](#switching)

## Performing side effects with signals

Most signals start out "cold," which means that they will not do any work until
[subscription](#subscription).

Upon subscription, a signal or its [subscribers][Subscription] can perform _side
effects_, like logging to the console, making a network request, updating the
user interface, etc.

Side effects can also be [injected](#injecting-effects) into a signal, where
they won't be performed immediately, but will instead take effect with each
subscription later.

### Subscription

The [-subscribe…][RACSignal] methods give you access to the current and future values in a signal:

```objc
RACSignal *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence.signal;

// Outputs: A B C D E F G H I
[letters subscribeNext:^(NSString *x) {
    NSLog(@"%@", x);
}];
```

For a cold signal, side effects will be performed once _per subscription_:

```objc
__block unsigned subscriptions = 0;

RACSignal *loggingSignal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
    subscriptions++;
    [subscriber sendCompleted];
    return nil;
}];

// Outputs:
// subscription 1
[loggingSignal subscribeCompleted:^{
    NSLog(@"subscription %u", subscriptions);
}];

// Outputs:
// subscription 2
[loggingSignal subscribeCompleted:^{
    NSLog(@"subscription %u", subscriptions);
}];
```

This behavior can be changed using a [connection][Connections].

### Injecting effects

The [-do…][RACSignal+Operations] methods add side effects to a signal without actually
subscribing to it:

```objc
__block unsigned subscriptions = 0;

RACSignal *loggingSignal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
    subscriptions++;
    [subscriber sendCompleted];
    return nil;
}];

// Does not output anything yet
loggingSignal = [loggingSignal doCompleted:^{
    NSLog(@"about to complete subscription %u", subscriptions);
}];

// Outputs:
// about to complete subscription 1
// subscription 1
[loggingSignal subscribeCompleted:^{
    NSLog(@"subscription %u", subscriptions);
}];
```

## Transforming streams

These operators transform a single stream into a new stream.

### Mapping

The [-map:][RACStream] method is used to transform the values in a stream, and
create a new stream with the results:

```objc
RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence;

// Contains: AA BB CC DD EE FF GG HH II
RACSequence *mapped = [letters map:^(NSString *value) {
    return [value stringByAppendingString:value];
}];
```

### Filtering

The [-filter:][RACStream] method uses a block to test each value, including it
into the resulting stream only if the test passes:

```objc
RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;

// Contains: 2 4 6 8
RACSequence *filtered = [numbers filter:^ BOOL (NSString *value) {
    return (value.intValue % 2) == 0;
}];
```

## Combining streams

These operators combine multiple streams into a single new stream.

### Concatenating

The [-concat:][RACStream] method appends one stream's values to another:

```objc
RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence;
RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;

// Contains: A B C D E F G H I 1 2 3 4 5 6 7 8 9
RACSequence *concatenated = [letters concat:numbers];
```

### Flattening

The [-flatten][RACStream] operator is applied to a stream-of-streams, and
combines their values into a single new stream.

Sequences are [concatenated](#concatenating):

```objc
RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence;
RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;
RACSequence *sequenceOfSequences = @[ letters, numbers ].rac_sequence;

// Contains: A B C D E F G H I 1 2 3 4 5 6 7 8 9
RACSequence *flattened = [sequenceOfSequences flatten];
```

Signals are [merged](#merging):

```objc
RACSubject *letters = [RACSubject subject];
RACSubject *numbers = [RACSubject subject];
RACSignal *signalOfSignals = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
    [subscriber sendNext:letters];
    [subscriber sendNext:numbers];
    [subscriber sendCompleted];
    return nil;
}];

RACSignal *flattened = [signalOfSignals flatten];

// Outputs: A 1 B C 2
[flattened subscribeNext:^(NSString *x) {
    NSLog(@"%@", x);
}];

[letters sendNext:@"A"];
[numbers sendNext:@"1"];
[letters sendNext:@"B"];
[letters sendNext:@"C"];
[numbers sendNext:@"2"];
```

### Mapping and flattening

[Flattening](#flattening) isn't that interesting on its own, but understanding
how it works is important for [-flattenMap:][RACStream].

`-flattenMap:` is used to transform each of a stream's values into _a new
stream_. Then, all of the streams returned will be flattened down into a single
stream. In other words, it's [-map:](#mapping) followed by [-flatten](#flattening).

This can be used to extend or edit sequences:

```objc
RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;

// Contains: 1 1 2 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9
RACSequence *extended = [numbers flattenMap:^(NSString *num) {
    return @[ num, num ].rac_sequence;
}];

// Contains: 1_ 3_ 5_ 7_ 9_
RACSequence *edited = [numbers flattenMap:^(NSString *num) {
    if (num.intValue % 2 == 0) {
        return [RACSequence empty];
    } else {
        NSString *newNum = [num stringByAppendingString:@"_"];
        return [RACSequence return:newNum]; 
    }
}];
```

Or create multiple signals of work which are automatically recombined:

```objc
RACSignal *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence.signal;

[[letters
    flattenMap:^(NSString *letter) {
        return [database saveEntriesForLetter:letter];
    }]
    subscribeCompleted:^{
        NSLog(@"All database entries saved successfully.");
    }];
```

## Combining signals

These operators combine multiple signals into a single new [RACSignal][].

### Sequencing

[-then:][RACSignal+Operations] starts the original signal,
waits for it to complete, and then only forwards the values from a new signal:

```objc
RACSignal *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence.signal;

// The new signal only contains: 1 2 3 4 5 6 7 8 9
//
// But when subscribed to, it also outputs: A B C D E F G H I
RACSignal *sequenced = [[letters
    doNext:^(NSString *letter) {
        NSLog(@"%@", letter);
    }]
    then:^{
        return [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence.signal;
    }];
```

This is most useful for executing all the side effects of one signal, then
starting another, and only returning the second signal's values.

### Merging

The [+merge:][RACSignal+Operations] method will forward the values from many
signals into a single stream, as soon as those values arrive:

```objc
RACSubject *letters = [RACSubject subject];
RACSubject *numbers = [RACSubject subject];
RACSignal *merged = [RACSignal merge:@[ letters, numbers ]];

// Outputs: A 1 B C 2
[merged subscribeNext:^(NSString *x) {
    NSLog(@"%@", x);
}];

[letters sendNext:@"A"];
[numbers sendNext:@"1"];
[letters sendNext:@"B"];
[letters sendNext:@"C"];
[numbers sendNext:@"2"];
```

### Combining latest values

The [+combineLatest:][RACSignal+Operations] and `+combineLatest:reduce:` methods
will watch multiple signals for changes, and then send the latest values from
_all_ of them when a change occurs:

```objc
RACSubject *letters = [RACSubject subject];
RACSubject *numbers = [RACSubject subject];
RACSignal *combined = [RACSignal
    combineLatest:@[ letters, numbers ]
    reduce:^(NSString *letter, NSString *number) {
        return [letter stringByAppendingString:number];
    }];

// Outputs: B1 B2 C2 C3
[combined subscribeNext:^(id x) {
    NSLog(@"%@", x);
}];

[letters sendNext:@"A"];
[letters sendNext:@"B"];
[numbers sendNext:@"1"];
[numbers sendNext:@"2"];
[letters sendNext:@"C"];
[numbers sendNext:@"3"];
```

Note that the combined signal will only send its first value when all of the
inputs have sent at least one. In the example above, `@"A"` was never
forwarded because `numbers` had not sent a value yet.

### Switching

The [-switchToLatest][RACSignal+Operations] operator is applied to
a signal-of-signals, and always forwards the values from the latest signal:

```objc
RACSubject *letters = [RACSubject subject];
RACSubject *numbers = [RACSubject subject];
RACSubject *signalOfSignals = [RACSubject subject];

RACSignal *switched = [signalOfSignals switchToLatest];

// Outputs: A B 1 D
[switched subscribeNext:^(NSString *x) {
    NSLog(@"%@", x);
}];

[signalOfSignals sendNext:letters];
[letters sendNext:@"A"];
[letters sendNext:@"B"];

[signalOfSignals sendNext:numbers];
[letters sendNext:@"C"];
[numbers sendNext:@"1"];

[signalOfSignals sendNext:letters];
[numbers sendNext:@"2"];
[letters sendNext:@"D"];
```

[Connections]: FrameworkOverview.md#connections
[RACSequence]: ../../ReactiveCocoa/Objective-C/RACSequence.h
[RACSignal]: ../../ReactiveCocoa/Objective-C/RACSignal.h
[RACSignal+Operations]: ../../ReactiveCocoa/Objective-C/RACSignal+Operations.h
[RACStream]: ../../ReactiveCocoa/Objective-C/RACStream.h
[Sequences]: FrameworkOverview.md#sequences
[Signals]: FrameworkOverview.md#signals
[Streams]: FrameworkOverview.md#streams
[Subscription]: FrameworkOverview.md#subscription


# Design Guidelines

This document contains guidelines for projects that want to make use of
ReactiveCocoa. The content here is heavily inspired by the [Rx Design
Guidelines](http://blogs.msdn.com/b/rxteam/archive/2010/10/28/rx-design-guidelines.aspx).

This document assumes basic familiarity
with the features of ReactiveCocoa. The [Framework Overview][] is a better
resource for getting up to speed on the functionality provided by RAC.

**[The RACSequence contract](#the-racsequence-contract)**

 1. [Evaluation occurs lazily by default](#evaluation-occurs-lazily-by-default)
 1. [Evaluation blocks the caller](#evaluation-blocks-the-caller)
 1. [Side effects occur only once](#side-effects-occur-only-once)

**[The RACSignal contract](#the-racsignal-contract)**

 1. [Signal events are serialized](#signal-events-are-serialized)
 1. [Subscription will always occur on a scheduler](#subscription-will-always-occur-on-a-scheduler)
 1. [Errors are propagated immediately](#errors-are-propagated-immediately)
 1. [Side effects occur for each subscription](#side-effects-occur-for-each-subscription)
 1. [Subscriptions are automatically disposed upon completion or error](#subscriptions-are-automatically-disposed-upon-completion-or-error)
 1. [Disposal cancels in-progress work and cleans up resources](#disposal-cancels-in-progress-work-and-cleans-up-resources)

**[Best practices](#best-practices)**

 1. [Use descriptive declarations for methods and properties that return a signal](#use-descriptive-declarations-for-methods-and-properties-that-return-a-signal)
 1. [Indent stream operations consistently](#indent-stream-operations-consistently)
 1. [Use the same type for all the values of a stream](#use-the-same-type-for-all-the-values-of-a-stream)
 1. [Avoid retaining streams for too long](#avoid-retaining-streams-for-too-long)
 1. [Process only as much of a stream as needed](#process-only-as-much-of-a-stream-as-needed)
 1. [Deliver signal events onto a known scheduler](#deliver-signal-events-onto-a-known-scheduler)
 1. [Switch schedulers in as few places as possible](#switch-schedulers-in-as-few-places-as-possible)
 1. [Make the side effects of a signal explicit](#make-the-side-effects-of-a-signal-explicit)
 1. [Share the side effects of a signal by multicasting](#share-the-side-effects-of-a-signal-by-multicasting)
 1. [Debug streams by giving them names](#debug-streams-by-giving-them-names)
 1. [Avoid explicit subscriptions and disposal](#avoid-explicit-subscriptions-and-disposal)
 1. [Avoid using subjects when possible](#avoid-using-subjects-when-possible)

**[Implementing new operators](#implementing-new-operators)**

 1. [Prefer building on RACStream methods](#prefer-building-on-racstream-methods)
 1. [Compose existing operators when possible](#compose-existing-operators-when-possible)
 1. [Avoid introducing concurrency](#avoid-introducing-concurrency)
 1. [Cancel work and clean up all resources in a disposable](#cancel-work-and-clean-up-all-resources-in-a-disposable)
 1. [Do not block in an operator](#do-not-block-in-an-operator)
 1. [Avoid stack overflow from deep recursion](#avoid-stack-overflow-from-deep-recursion)

## The RACSequence contract

[RACSequence][] is a _pull-driven_ stream. Sequences behave similarly to
built-in collections, but with a few unique twists.

### Evaluation occurs lazily by default

Sequences are evaluated lazily by default. For example, in this sequence:

```objc
NSArray *strings = @[ @"A", @"B", @"C" ];
RACSequence *sequence = [strings.rac_sequence map:^(NSString *str) {
    return [str stringByAppendingString:@"_"];
}];
```

… no string appending is actually performed until the values of the sequence are
needed. Accessing `sequence.head` will perform the concatenation of `A_`,
accessing `sequence.tail.head` will perform the concatenation of `B_`, and so
on.

This generally avoids performing unnecessary work (since values that are never
used are never calculated), but means that sequence processing [should be
limited only to what's actually
needed](#process-only-as-much-of-a-stream-as-needed).

Once evaluated, the values in a sequence are memoized and do not need to be
recalculated. Accessing `sequence.head` multiple times will only do the work of
one string concatenation.

If lazy evaluation is undesirable – for instance, because limiting memory usage
is more important than avoiding unnecessary work – the
[eagerSequence][RACSequence] property can be used to force a sequence (and any
sequences derived from it afterward) to evaluate eagerly.

### Evaluation blocks the caller

Regardless of whether a sequence is lazy or eager, evaluation of any part of
a sequence will block the calling thread until completed. This is necessary
because values must be synchronously retrieved from a sequence.

If evaluating a sequence is expensive enough that it might block the thread for
a significant amount of time, consider creating a signal with
[-signalWithScheduler:][RACSequence] and using that instead.

### Side effects occur only once

When the block passed to a sequence operator involves side effects, it is
important to realize that those side effects will only occur once per value
– namely, when the value is evaluated:

```objc
NSArray *strings = @[ @"A", @"B", @"C" ];
RACSequence *sequence = [strings.rac_sequence map:^(NSString *str) {
    NSLog(@"%@", str);
    return [str stringByAppendingString:@"_"];
}];

// Logs "A" during this call.
NSString *concatA = sequence.head;

// Logs "B" during this call.
NSString *concatB = sequence.tail.head;

// Does not log anything.
NSString *concatB2 = sequence.tail.head;

RACSequence *derivedSequence = [sequence map:^(NSString *str) {
    return [@"_" stringByAppendingString:str];
}];

// Still does not log anything, because "B_" was already evaluated, and the log
// statement associated with it will never be re-executed.
NSString *concatB3 = derivedSequence.tail.head;
```

## The RACSignal contract

[RACSignal][] is a _push-driven_ stream with a focus on asynchronous event
delivery through _subscriptions_. For more information about signals and
subscriptions, see the [Framework Overview][].

### Signal events are serialized

A signal may choose to deliver its events on any thread. Consecutive events are
even allowed to arrive on different threads or schedulers, unless explicitly
[delivered onto a particular
scheduler](#deliver-signal-events-onto-a-known-scheduler).

However, RAC guarantees that no two signal events will ever arrive concurrently.
While an event is being processed, no other events will be delivered. The
senders of any other events will be forced to wait until the current event has
been handled.

Most notably, this means that the blocks passed to
[-subscribeNext:error:completed:][RACSignal] do not need to be synchronized with
respect to each other, because they will never be invoked simultaneously.

### Subscription will always occur on a scheduler

To ensure consistent behavior for the `+createSignal:` and `-subscribe:`
methods, each [RACSignal][] subscription is guaranteed to take place on
a valid [RACScheduler][].

If the subscriber's thread already has a [+currentScheduler][RACScheduler],
scheduling takes place immediately; otherwise, scheduling occurs as soon as
possible on a background scheduler. Note that the main thread is always
associated with the [+mainThreadScheduler][RACScheduler], so subscription will
always be immediate there.

See the documentation for [-subscribe:][RACSignal] for more information.

### Errors are propagated immediately

In RAC, `error` events have exception semantics. When an error is sent on
a signal, it will be immediately forwarded to all dependent signals, causing the
entire chain to terminate.

[Operators][RACSignal+Operations] whose primary purpose is to change
error-handling behavior – like `-catch:`, `-catchTo:`, or `-materialize` – are
obviously not subject to this rule.

### Side effects occur for each subscription

Each new subscription to a [RACSignal][] will trigger its side effects. This
means that any side effects will happen as many times as subscriptions to the
signal itself.

Consider this example:
```objc
__block int aNumber = 0;

// Signal that will have the side effect of incrementing `aNumber` block 
// variable for each subscription before sending it.
RACSignal *aSignal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
	aNumber++;
	[subscriber sendNext:@(aNumber)];
	[subscriber sendCompleted];
	return nil;
}];

// This will print "subscriber one: 1"
[aSignal subscribeNext:^(id x) {
	NSLog(@"subscriber one: %@", x);
}];

// This will print "subscriber two: 2"
[aSignal subscribeNext:^(id x) {
	NSLog(@"subscriber two: %@", x);
}];
```

Side effects are repeated for each subscription. The same applies to
[stream][RACStream] and [signal][RACSignal+Operations] operators:

```objc
__block int missilesToLaunch = 0;

// Signal that will have the side effect of changing `missilesToLaunch` on
// subscription.
RACSignal *processedSignal = [[RACSignal
    return:@"missiles"]
	map:^(id x) {
		missilesToLaunch++;
		return [NSString stringWithFormat:@"will launch %d %@", missilesToLaunch, x];
	}];

// This will print "First will launch 1 missiles"
[processedSignal subscribeNext:^(id x) {
	NSLog(@"First %@", x);
}];

// This will print "Second will launch 2 missiles"
[processedSignal subscribeNext:^(id x) {
	NSLog(@"Second %@", x);
}];
```

To suppress this behavior and have multiple subscriptions to a signal execute
its side effects only once, a signal can be 
[multicasted](#share-the-side-effects-of-a-signal-by-multicasting).

Side effects can be insidious and produce problems that are difficult to
diagnose. For this reason it is suggested to 
[make side effects explicit](#make-the-side-effects-of-a-signal-explicit) when 
possible.

### Subscriptions are automatically disposed upon completion or error

When a [subscriber][RACSubscriber] is sent a `completed` or `error` event, the
associated subscription will automatically be disposed. This behavior usually
eliminates the need to manually dispose of subscriptions.

See the [Memory Management][] document for more information about signal
lifetime.

### Disposal cancels in-progress work and cleans up resources

When a subscription is disposed, manually or automatically, any in-progress or
outstanding work associated with that subscription is gracefully cancelled as
soon as possible, and any resources associated with the subscription are cleaned
up.

Disposing of the subscription to a signal representing a file upload, for
example, would cancel any in-flight network request, and free the file data from
memory.

## Best practices

The following recommendations are intended to help keep RAC-based code
predictable, understandable, and performant.

They are, however, only guidelines. Use best judgement when determining whether
to apply the recommendations here to a given piece of code.

### Use descriptive declarations for methods and properties that return a signal

When a method or property has a return type of [RACSignal][], it can be
difficult to understand the signal's semantics at a glance.

There are three key questions that can inform a declaration:

 1. Is the signal _hot_ (already activated by the time it's returned to the
    caller) or _cold_ (activated when subscribed to)?
 1. Will the signal include zero, one, or more values?
 1. Does the signal have side effects?

**Hot signals without side effects** should typically be properties instead of
methods. The use of a property indicates that no initialization is needed before
subscribing to the signal's events, and that additional subscribers will not
change the semantics. Signal properties should usually be named after events
(e.g., `textChanged`).

**Cold signals without side effects** should be returned from methods that have
noun-like names (e.g., `-currentText`). A method declaration indicates that the
signal might not be kept around, hinting that work is performed at the time of
subscription. If the signal sends multiple values, the noun should be pluralized
(e.g., `-currentModels`).

**Signals with side effects** should be returned from methods that have
verb-like names (e.g., `-logIn`). The verb indicates that the method is not
idempotent and that callers must be careful to call it only when the side
effects are desired. If the signal will send one or more values, include a noun
that describes them (e.g., `-loadConfiguration`, `-fetchLatestEvents`).

### Indent stream operations consistently

It's easy for stream-heavy code to become very dense and confusing if not
properly formatted. Use consistent indentation to highlight where chains of
streams begin and end.

When invoking a single method upon a stream, no additional indentation is
necessary (block arguments aside):

```objc
RACStream *result = [stream startWith:@0];

RACStream *result2 = [stream map:^(NSNumber *value) {
    return @(value.integerValue + 1);
}];
```

When transforming the same stream multiple times, ensure that all of the
steps are aligned. Complex operators like [+zip:reduce:][RACStream] or
[+combineLatest:reduce:][RACSignal+Operations] may be split over multiple lines
for readability:

```objc
RACStream *result = [[[RACStream
    zip:@[ firstStream, secondStream ]
    reduce:^(NSNumber *first, NSNumber *second) {
        return @(first.integerValue + second.integerValue);
    }]
    filter:^ BOOL (NSNumber *value) {
        return value.integerValue >= 0;
    }]
    map:^(NSNumber *value) {
        return @(value.integerValue + 1);
    }];
```

Of course, streams nested within block arguments should start at the natural
indentation of the block:

```objc
[[signal
    then:^{
        @strongify(self);

        return [[self
            doSomethingElse]
            catch:^(NSError *error) {
                @strongify(self);
                [self presentError:error];

                return [RACSignal empty];
            }];
    }]
    subscribeCompleted:^{
        NSLog(@"All done.");
    }];
```

### Use the same type for all the values of a stream

[RACStream][] (and, by extension, [RACSignal][] and [RACSequence][]) allows
streams to be composed of heterogenous objects, just like Cocoa collections do.
However, using different object types within the same stream complicates the use
of operators and
puts an additional burden on any consumers of that stream, who must be careful to
only invoke supported methods.

Whenever possible, streams should only contain objects of the same type.

### Avoid retaining streams for too long

Retaining any [RACStream][] longer than it's needed will cause any dependencies
to be retained as well, potentially keeping memory usage much higher than it
would be otherwise.

A [RACSequence][] should be retained only for as long as the `head` of the
sequence is needed. If the head will no longer be used, retain the `tail` of the
node instead of the node itself.

See the [Memory Management][] guide for more information on object lifetime.

### Process only as much of a stream as needed

As well as [consuming additional
memory](#avoid-retaining-streams-for-too-long), unnecessarily
keeping a stream or [RACSignal][] subscription alive can result in increased CPU
usage, as unnecessary work is performed for results that will never be used.

If only a certain number of values are needed from a stream, the
[-take:][RACStream] operator can be used to retrieve only that many values, and
then automatically terminate the stream immediately thereafter.

Operators like `-take:` and [-takeUntil:][RACSignal+Operations] automatically propagate cancellation
up the stack as well. If nothing else needs the rest of the values, any
dependencies will be terminated too, potentially saving a significant amount of
work.

### Deliver signal events onto a known scheduler

When a signal is returned from a method, or combined with such a signal, it can
be difficult to know which thread events will be delivered upon. Although
events are [guaranteed to be serial](#signal-events-are-serialized), sometimes
stronger guarantees are needed, like when performing UI updates (which must
occur on the main thread).

Whenever such a guarantee is important, the [-deliverOn:][RACSignal+Operations]
operator should be used to force a signal's events to arrive on a specific
[RACScheduler][].

### Switch schedulers in as few places as possible

Notwithstanding the above, events should only be delivered to a specific
[scheduler][RACScheduler] when absolutely necessary. Switching schedulers can
introduce unnecessary delays and cause an increase in CPU load.

Generally, the use of [-deliverOn:][RACSignal+Operations] should be restricted
to the end of a signal chain – e.g., before subscription, or before the values
are bound to a property.

### Make the side effects of a signal explicit

As much as possible, [RACSignal][] side effects should be avoided, because
subscribers may find the [behavior of side
effects](#side-effects-occur-for-each-subscription) unexpected.

However, because Cocoa is predominantly imperative, it is sometimes useful to
perform side effects when signal events occur. Although most [RACStream][] and
[RACSignal][RACSignal+Operations] operators accept arbitrary blocks (which can
contain side effects), the use of `-doNext:`, `-doError:`, and `-doCompleted:`
will make side effects more explicit and self-documenting:

```objc
NSMutableArray *nexts = [NSMutableArray array];
__block NSError *receivedError = nil;
__block BOOL success = NO;

RACSignal *bookkeepingSignal = [[[valueSignal
    doNext:^(id x) {
        [nexts addObject:x];
    }]
    doError:^(NSError *error) {
        receivedError = error;
    }]
    doCompleted:^{
        success = YES;
    }];

RAC(self, value) = bookkeepingSignal;
```

### Share the side effects of a signal by multicasting

[Side effects occur for each
subscription](#side-effects-occur-for-each-subscription) by default, but there
are certain situations where side effects should only occur once – for example,
a network request typically should not be repeated when a new subscriber is
added.

The `-publish` and `-multicast:` operators of [RACSignal][RACSignal+Operations]
allow a single subscription to be shared to any number of subscribers by using
a [RACMulticastConnection][]:

```objc
// This signal starts a new request on each subscription.
RACSignal *networkRequest = [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
    AFHTTPRequestOperation *operation = [client
        HTTPRequestOperationWithRequest:request
        success:^(AFHTTPRequestOperation *operation, id response) {
            [subscriber sendNext:response];
            [subscriber sendCompleted];
        }
        failure:^(AFHTTPRequestOperation *operation, NSError *error) {
            [subscriber sendError:error];
        }];

    [client enqueueHTTPRequestOperation:operation];
    return [RACDisposable disposableWithBlock:^{
        [operation cancel];
    }];
}];

// Starts a single request, no matter how many subscriptions `connection.signal`
// gets. This is equivalent to the -replay operator, or similar to
// +startEagerlyWithScheduler:block:.
RACMulticastConnection *connection = [networkRequest multicast:[RACReplaySubject subject]];
[connection connect];

[connection.signal subscribeNext:^(id response) {
    NSLog(@"subscriber one: %@", response);
}];

[connection.signal subscribeNext:^(id response) {
    NSLog(@"subscriber two: %@", response);
}];
```

### Debug streams by giving them names

Every [RACStream][] has a `name` property to assist with debugging. A stream's
`-description` includes its name, and all operators provided by RAC will
automatically add to the name. This usually makes it possible to identify
a stream from its default name alone.

For example, this snippet:

```objc
RACSignal *signal = [[[RACObserve(self, username) 
    distinctUntilChanged] 
    take:3] 
    filter:^(NSString *newUsername) {
        return [newUsername isEqualToString:@"joshaber"];
    }];

NSLog(@"%@", signal);
```

… would log a name similar to `[[[RACObserve(self, username)] -distinctUntilChanged]
-take: 3] -filter:`.

Names can also be manually applied by using [-setNameWithFormat:][RACStream].

[RACSignal][] also offers `-logNext`, `-logError`,
`-logCompleted`, and `-logAll` methods, which will automatically log signal
events as they occur, and include the name of the signal in the messages. This
can be used to conveniently inspect a signal in real-time.

### Avoid explicit subscriptions and disposal

Although [-subscribeNext:error:completed:][RACSignal] and its variants are the
most basic way to process a signal, their use can complicate code by
being less declarative, encouraging the use of side effects, and potentially
duplicating built-in functionality.

Likewise, explicit use of the [RACDisposable][] class can quickly lead to
a rat's nest of resource management and cleanup code.

There are almost always higher-level patterns that can be used instead of manual
subscriptions and disposal:

 * The [RAC()][RAC] or [RACChannelTo()][RACChannelTo] macros can be used to bind
   a signal to a property, instead of performing manual updates when changes
   occur.
 * The [-rac_liftSelector:withSignals:][NSObject+RACLifting] method can be used
   to automatically invoke a selector when one or more signals fire.
 * Operators like [-takeUntil:][RACSignal+Operations] can be used to
   automatically dispose of a subscription when an event occurs (like a "Cancel"
   button being pressed in the UI).

Generally, the use of built-in [stream][RACStream] and
[signal][RACSignal+Operations] operators will lead to simpler and less
error-prone code than replicating the same behaviors in a subscription callback.

### Avoid using subjects when possible

[Subjects][] are a powerful tool for bridging imperative code
into the world of signals, but, as the "mutable variables" of RAC, they can
quickly lead to complexity when overused.

Since they can be manipulated from anywhere, at any time, subjects often break
the linear flow of stream processing and make logic much harder to follow. They
also don't support meaningful
[disposal](#disposal-cancels-in-progress-work-and-cleans-up-resources), which
can result in unnecessary work.

Subjects can usually be replaced with other patterns from ReactiveCocoa:

 * Instead of feeding initial values into a subject, consider generating the
   values in a [+createSignal:][RACSignal] block instead.
 * Instead of delivering intermediate results to a subject, try combining the
   output of multiple signals with operators like
   [+combineLatest:][RACSignal+Operations] or [+zip:][RACStream].
 * Instead of using subjects to share results with multiple subscribers,
   [multicast](#share-the-side-effects-of-a-signal-by-multicasting) a base
   signal instead.
 * Instead of implementing an action method which simply controls a subject, use
   a [command][RACCommand] or
   [-rac_signalForSelector:][NSObject+RACSelectorSignal] instead.

When subjects _are_ necessary, they should almost always be the "base" input
for a signal chain, not used in the middle of one.

## Implementing new operators

RAC provides a long list of built-in operators for [streams][RACStream] and
[signals][RACSignal+Operations] that should cover most use cases; however, RAC
is not a closed system. It's entirely valid to implement additional operators
for specialized uses, or for consideration in ReactiveCocoa itself.

Implementing a new operator requires a careful attention to detail and a focus
on simplicity, to avoid introducing bugs into the calling code.

These guidelines cover some of the common pitfalls and help preserve the
expected API contracts.

### Prefer building on RACStream methods

[RACStream][] offers a simpler interface than [RACSequence][] and [RACSignal][],
and all stream operators are automatically applicable to sequences and signals
as well.

For these reasons, new operators should be implemented using only [RACStream][]
methods whenever possible. The minimal required methods of the class, including
`-bind:`, `-zipWith:`, and `-concat:`, are quite powerful, and many tasks can
be accomplished without needing anything else.

If a new [RACSignal][] operator needs to handle `error` and `completed` events,
consider using the [-materialize][RACSignal+Operations] method to bring the
events into the stream. All of the events of a materialized signal can be
manipulated by stream operators, which helps minimize the use of non-stream
operators.

### Compose existing operators when possible

Considerable thought has been put into the operators provided by RAC, and they
have been validated through automated tests and through their real world use in
other projects. An operator that has been written from scratch may not be as
robust, or might not handle a special case that the built-in operators are aware
of.

To minimize duplication and possible bugs, use the provided operators as much as
possible in a custom operator implementation. Generally, there should be very
little code written from scratch.

### Avoid introducing concurrency

Concurrency is an extremely common source of bugs in programming. To minimize
the potential for deadlocks and race conditions, operators should not
concurrently perform their work.

Callers always have the ability to subscribe or deliver events on a specific
[RACScheduler][], and RAC offers powerful ways to [parallelize
work][Parallelizing Independent Work] without making operators unnecessarily
complex.

### Cancel work and clean up all resources in a disposable

When implementing a signal with the [+createSignal:][RACSignal] method, the
provided block is expected to return a [RACDisposable][]. This disposable
should:

 * As soon as it is convenient, gracefully cancel any in-progress work that was
   started by the signal.
 * Immediately dispose of any subscriptions to other signals, thus triggering
   their cancellation and cleanup code as well.
 * Release any memory or other resources that were allocated by the signal.

This helps fulfill [the RACSignal
contract](#disposal-cancels-in-progress-work-and-cleans-up-resources).

### Do not block in an operator

Stream operators should return a new stream more-or-less immediately. Any work
that the operator needs to perform should be part of evaluating the new stream,
_not_ part of the operator invocation itself.

```objc
// WRONG!
- (RACSequence *)map:(id (^)(id))block {
    RACSequence *result = [RACSequence empty];
    for (id obj in self) {
        id mappedObj = block(obj);
        result = [result concat:[RACSequence return:mappedObj]];
    }

    return result;
}

// Right!
- (RACSequence *)map:(id (^)(id))block {
    return [self flattenMap:^(id obj) {
        id mappedObj = block(obj);
        return [RACSequence return:mappedObj];
    }];
}
```

This guideline can be safely ignored when the purpose of an operator is to
synchronously retrieve one or more values from a stream (like
[-first][RACSignal+Operations]).

### Avoid stack overflow from deep recursion

Any operator that might recurse indefinitely should use the
`-scheduleRecursiveBlock:` method of [RACScheduler][]. This method will
transform recursion into iteration instead, preventing a stack overflow.

For example, this would be an incorrect implementation of
[-repeat][RACSignal+Operations], due to its potential to overflow the call stack
and cause a crash:

```objc
- (RACSignal *)repeat {
    return [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

        __block void (^resubscribe)(void) = ^{
            RACDisposable *disposable = [self subscribeNext:^(id x) {
                [subscriber sendNext:x];
            } error:^(NSError *error) {
                [subscriber sendError:error];
            } completed:^{
                resubscribe();
            }];

            [compoundDisposable addDisposable:disposable];
        };

        return compoundDisposable;
    }];
}
```

By contrast, this version will avoid a stack overflow:

```objc
- (RACSignal *)repeat {
    return [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

        RACScheduler *scheduler = RACScheduler.currentScheduler ?: [RACScheduler scheduler];
        RACDisposable *disposable = [scheduler scheduleRecursiveBlock:^(void (^reschedule)(void)) {
            RACDisposable *disposable = [self subscribeNext:^(id x) {
                [subscriber sendNext:x];
            } error:^(NSError *error) {
                [subscriber sendError:error];
            } completed:^{
                reschedule();
            }];

            [compoundDisposable addDisposable:disposable];
        }];

        [compoundDisposable addDisposable:disposable];
        return compoundDisposable;
    }];
}
```

[Framework Overview]: FrameworkOverview.md
[Memory Management]: MemoryManagement.md
[NSObject+RACLifting]: ../../ReactiveCocoa/Objective-C/NSObject+RACLifting.h
[NSObject+RACSelectorSignal]: ../../ReactiveCocoa/Objective-C/NSObject+RACSelectorSignal.h
[RAC]: ../../ReactiveCocoa/Objective-C/RACSubscriptingAssignmentTrampoline.h
[RACChannelTo]: ../../ReactiveCocoa/Objective-C/RACKVOChannel.h
[RACCommand]: ../../ReactiveCocoa/Objective-C/RACCommand.h
[RACDisposable]: ../../ReactiveCocoa/Objective-C/RACDisposable.h
[RACEvent]: ../../ReactiveCocoa/Objective-C/RACEvent.h
[RACMulticastConnection]: ../../ReactiveCocoa/Objective-C/RACMulticastConnection.h
[RACObserve]: ../../ReactiveCocoa/Objective-C/NSObject+RACPropertySubscribing.h
[RACScheduler]: ../../ReactiveCocoa/Objective-C/RACScheduler.h
[RACSequence]: ../../ReactiveCocoa/Objective-C/RACSequence.h
[RACSignal]: ../../ReactiveCocoa/Objective-C/RACSignal.h
[RACSignal+Operations]: ../../ReactiveCocoa/Objective-C/RACSignal+Operations.h
[RACStream]: ../../ReactiveCocoa/Objective-C/RACStream.h
[RACSubscriber]: ../../ReactiveCocoa/Objective-C/RACSubscriber.h
[Subjects]: FrameworkOverview.md#subjects
[Parallelizing Independent Work]: ../README.md#parallelizing-independent-work



# Framework Overview

This document contains a high-level description of the different components
within the ReactiveCocoa framework, and an attempt to explain how they work
together and divide responsibilities. This is meant to be a starting point for
learning about new modules and finding more specific documentation.

For examples and help understanding how to use RAC, see the [README][] or
the [Design Guidelines][].

## Streams

A **stream**, represented by the [RACStream][] abstract class, is any series of
object values.

Values may be available immediately or in the future, but must be retrieved
sequentially. There is no way to retrieve the second value of a stream without
evaluating or waiting for the first value.

Streams are [monads][]. Among other things, this allows complex operations to be
built on a few basic primitives (`-bind:` in particular). [RACStream][] also
implements the equivalent of the [Monoid][] and [MonadZip][] typeclasses from
[Haskell][].

[RACStream][] isn't terribly useful on its own. Most streams are treated as
[signals](#signals) or [sequences](#sequences) instead.

## Signals

A **signal**, represented by the [RACSignal][] class, is a _push-driven_
[stream](#streams).

Signals generally represent data that will be delivered in the future. As work
is performed or data is received, values are _sent_ on the signal, which pushes
them out to any subscribers. Users must [subscribe](#subscription) to a signal
in order to access its values.

Signals send three different types of events to their subscribers:

 * The **next** event provides a new value from the stream. [RACStream][]
   methods only operate on events of this type. Unlike Cocoa collections, it is
   completely valid for a signal to include `nil`.
 * The **error** event indicates that an error occurred before the signal could
   finish. The event may include an `NSError` object that indicates what went
   wrong. Errors must be handled specially – they are not included in the
   stream's values.
 * The **completed** event indicates that the signal finished successfully, and
   that no more values will be added to the stream. Completion must be handled
   specially – it is not included in the stream of values.

The lifetime of a signal consists of any number of `next` events, followed by
one `error` or `completed` event (but not both).

### Subscription

A **subscriber** is anything that is waiting or capable of waiting for events
from a [signal](#signals). Within RAC, a subscriber is represented as any object
that conforms to the [RACSubscriber][] protocol.

A **subscription** is created through any call to
[-subscribeNext:error:completed:][RACSignal], or one of the corresponding
convenience methods. Technically, most [RACStream][] and
[RACSignal][RACSignal+Operations] operators create subscriptions as well, but
these intermediate subscriptions are usually an implementation detail.

Subscriptions [retain their signals][Memory Management], and are automatically
disposed of when the signal completes or errors. Subscriptions can also be
[disposed of manually](#disposables).

### Subjects

A **subject**, represented by the [RACSubject][] class, is a [signal](#signals)
that can be manually controlled.

Subjects can be thought of as the "mutable" variant of a signal, much like
`NSMutableArray` is for `NSArray`. They are extremely useful for bridging
non-RAC code into the world of signals.

For example, instead of handling application logic in block callbacks, the
blocks can simply send events to a shared subject instead. The subject can then
be returned as a [RACSignal][], hiding the implementation detail of the
callbacks.

Some subjects offer additional behaviors as well. In particular,
[RACReplaySubject][] can be used to buffer events for future
[subscribers](#subscription), like when a network request finishes before
anything is ready to handle the result.

### Commands

A **command**, represented by the [RACCommand][] class, creates and subscribes
to a signal in response to some action. This makes it easy to perform
side-effecting work as the user interacts with the app.

Usually the action triggering a command is UI-driven, like when a button is
clicked. Commands can also be automatically disabled based on a signal, and this
disabled state can be represented in a UI by disabling any controls associated
with the command.

On OS X, RAC adds a `rac_command` property to
[NSButton][NSButton+RACCommandSupport] for setting up these behaviors
automatically.

### Connections

A **connection**, represented by the [RACMulticastConnection][] class, is
a [subscription](#subscription) that is shared between any number of
subscribers.

[Signals](#signals) are _cold_ by default, meaning that they start doing work
_each_ time a new subscription is added. This behavior is usually desirable,
because it means that data will be freshly recalculated for each subscriber, but
it can be problematic if the signal has side effects or the work is expensive
(for example, sending a network request).

A connection is created through the `-publish` or `-multicast:` methods on
[RACSignal][RACSignal+Operations], and ensures that only one underlying
subscription is created, no matter how many times the connection is subscribed
to. Once connected, the connection's signal is said to be _hot_, and the
underlying subscription will remain active until _all_ subscriptions to the
connection are [disposed](#disposables).

## Sequences

A **sequence**, represented by the [RACSequence][] class, is a _pull-driven_
[stream](#streams).

Sequences are a kind of collection, similar in purpose to `NSArray`. Unlike
an array, the values in a sequence are evaluated _lazily_ (i.e., only when they
are needed) by default, potentially improving performance if only part of
a sequence is used. Just like Cocoa collections, sequences cannot contain `nil`.

Sequences are similar to [Clojure's sequences][seq] ([lazy-seq][] in particular), or
the [List][] type in [Haskell][].

RAC adds a `-rac_sequence` method to most of Cocoa's collection classes,
allowing them to be used as [RACSequences][RACSequence] instead.

## Disposables

The **[RACDisposable][]** class is used for cancellation and resource cleanup.

Disposables are most commonly used to unsubscribe from a [signal](#signals).
When a [subscription](#subscription) is disposed, the corresponding subscriber
will not receive _any_ further events from the signal. Additionally, any work
associated with the subscription (background processing, network requests, etc.)
will be cancelled, since the results are no longer needed.

For more information about cancellation, see the RAC [Design Guidelines][].

## Schedulers

A **scheduler**, represented by the [RACScheduler][] class, is a serial
execution queue for [signals](#signals) to perform work or deliver their results upon.

Schedulers are similar to Grand Central Dispatch queues, but schedulers support
cancellation (via [disposables](#disposables)), and always execute serially.
With the exception of the [+immediateScheduler][RACScheduler], schedulers do not
offer synchronous execution. This helps avoid deadlocks, and encourages the use
of [signal operators][RACSignal+Operations] instead of blocking work.

[RACScheduler][] is also somewhat similar to `NSOperationQueue`, but schedulers
do not allow tasks to be reordered or depend on one another.

## Value types

RAC offers a few miscellaneous classes for conveniently representing values in
a [stream](#streams):

 * **[RACTuple][]** is a small, constant-sized collection that can contain
   `nil` (represented by `RACTupleNil`). It is generally used to represent
   the combined values of multiple streams.
 * **[RACUnit][]** is a singleton "empty" value. It is used as a value in
   a stream for those times when more meaningful data doesn't exist.
 * **[RACEvent][]** represents any [signal event](#signals) as a single value.
   It is primarily used by the `-materialize` method of
   [RACSignal][RACSignal+Operations].

[Design Guidelines]: DesignGuidelines.md
[Haskell]: http://www.haskell.org
[lazy-seq]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/lazy-seq
[List]: https://downloads.haskell.org/~ghc/latest/docs/html/libraries/Data-List.html
[Memory Management]: MemoryManagement.md
[monads]: http://en.wikipedia.org/wiki/Monad_(functional_programming)
[Monoid]: http://downloads.haskell.org/~ghc/latest/docs/html/libraries/Data-Monoid.html
[MonadZip]: http://downloads.haskell.org/~ghc/latest/docs/html/libraries/Control-Monad-Zip.html
[NSButton+RACCommandSupport]: ../../ReactiveCocoa/Objective-C/NSButton+RACCommandSupport.h
[RACCommand]: ../../ReactiveCocoa/Objective-C/RACCommand.h
[RACDisposable]: ../../ReactiveCocoa/Objective-C/RACDisposable.h
[RACEvent]: ../../ReactiveCocoa/Objective-C/RACEvent.h
[RACMulticastConnection]: ../../ReactiveCocoa/Objective-C/RACMulticastConnection.h
[RACReplaySubject]: ../../ReactiveCocoa/Objective-C/RACReplaySubject.h
[RACScheduler]: ../../ReactiveCocoa/Objective-C/RACScheduler.h
[RACSequence]: ../../ReactiveCocoa/Objective-C/RACSequence.h
[RACSignal]: ../../ReactiveCocoa/Objective-C/RACSignal.h
[RACSignal+Operations]: ../../ReactiveCocoa/Objective-C/RACSignal+Operations.h
[RACStream]: ../../ReactiveCocoa/Objective-C/RACStream.h
[RACSubject]: ../../ReactiveCocoa/Objective-C/RACSubject.h
[RACSubscriber]: ../../ReactiveCocoa/Objective-C/RACSubscriber.h
[RACTuple]: ../../ReactiveCocoa/Objective-C/RACTuple.h
[RACUnit]: ../../ReactiveCocoa/Objective-C/RACUnit.h
[README]: README.md
[seq]: http://clojure.org/sequences


# Memory Management

ReactiveCocoa's memory management is quite complex, but the end result is that
**you don't need to retain signals in order to process them**.

If the framework required you to retain every signal, it'd be much more unwieldy
to use, especially for one-shot signals that are used like futures (e.g.,
network requests). You'd have to save any long-lived signal into a property, and
then also make sure to clear it out when you're done with it. Not fun.

## Subscribers

Before going any further, it's worth noting that
`subscribeNext:error:completed:` (and all variants thereof) create an _implicit_
subscriber using the given blocks. Any objects referenced from those blocks will
therefore be retained as part of the subscription. Just like any other object,
`self` won't be retained without a direct or indirect reference to it.

## Finite or Short-Lived Signals

The most important guideline to RAC memory management is that a **subscription
is automatically terminated upon completion or error, and the subscriber
removed**.

For example, if you have some code like this in your view controller:

```objc
self.disposable = [signal subscribeCompleted:^{
    doSomethingPossiblyInvolving(self);
}];
```

… the memory management will look something like the following:

```
view controller -> RACDisposable -> RACSignal -> RACSubscriber -> view controller
```

However, the `RACSignal -> RACSubscriber` relationship is torn down as soon as
`signal` finishes, breaking the retain cycle.

**This is often all you need**, because the lifetime of the `RACSignal` in
memory will naturally match the logical lifetime of the event stream.

## Infinite Signals

Infinite signals (or signals that live so long that they might as well be
infinite), however, will never tear down naturally. This is where disposables
shine.

**Disposing of a subscription will remove the associated subscriber**, and just
generally clean up any resources associated with that subscription. To that one
subscriber, it's just as if the signal had completed or errored, except no final
event is sent on the signal. All other subscribers will remain intact.

However, as a general rule of thumb, if you have to manually manage
a subscription's lifecycle, [there's probably a better way to do what you want][avoid-explicit-subscriptions-and-disposal].

## Signals Derived from `self`

There's still a bit of a tricky middle case here, though. Any time a signal's
lifetime is tied to the calling scope, you'll have a much harder cycle to break.

This commonly occurs when using `RACObserve()` on a key
path that's relative to `self`, and then applying a block that needs to capture
`self`.

The easiest answer here is just to **capture `self` weakly**:

```objc
__weak id weakSelf = self;
[RACObserve(self, username) subscribeNext:^(NSString *username) {
    id strongSelf = weakSelf;
    [strongSelf validateUsername];
}];
```

Or, after importing the included
[EXTScope.h](https://github.com/jspahrsummers/libextobjc/blob/master/extobjc/EXTScope.h)
header:

```objc
@weakify(self);
[RACObserve(self, username) subscribeNext:^(NSString *username) {
    @strongify(self);
    [self validateUsername];
}];
```

*(Replace `__weak` or `@weakify` with `__unsafe_unretained` or `@unsafeify`,
respectively, if the object doesn't support weak references.)*

However, [there's probably a better pattern you could use instead][avoid-explicit-subscriptions-and-disposal]. For
example, the above sample could perhaps be written like:

```objc
[self rac_liftSelector:@selector(validateUsername:) withSignals:RACObserve(self, username), nil];
```

or:

```objc
RACSignal *validated = [RACObserve(self, username) map:^(NSString *username) {
    // Put validation logic here.
    return @YES;
}];
```

As with infinite signals, there are generally ways you can avoid referencing
`self` (or any object) from blocks in a signal chain.

----

The above information is really all you should need in order to use
ReactiveCocoa effectively. However, there's one more point to address, just for
the technically curious or for anyone interested in contributing to RAC.

The design goal of "no retaining necessary" begs the question: how do we know
when a signal should be deallocated? What if it was just created, escaped an
autorelease pool, and hasn't been retained yet?

The real answer is _we don't_, BUT we can usually assume that the caller will
retain the signal within the current run loop iteration if they want to keep it.

Consequently:

 1. A created signal is automatically added to a global set of active signals.
 2. The signal will wait for a single pass of the main run loop, and then remove
    itself from the active set _if it has no subscribers_. Unless the signal was
    retained somehow, it would deallocate at this point.
 3. If something did subscribe in that run loop iteration, the signal stays in
    the set.
 4. Later, when all the subscribers are gone, step 2 is triggered again.

This could backfire if the run loop is spun recursively (like in a modal event
loop on OS X), but it makes the life of the framework consumer much easier for
most or all other cases.

[avoid-explicit-subscriptions-and-disposal]: DesignGuidelines.md#avoid-explicit-subscriptions-and-disposal







