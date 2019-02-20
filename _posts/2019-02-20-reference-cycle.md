---
layout: post
title:  "How to Avoid Reference Cycle in ARC"
categories: [Dev]
tag: [ios, arc]
author: Mikemike Zhu
---

{:toc}

## Abstract

Both Swift and Objective-C are using ARC (Auto Reference Counting) to manage the memory of the application. In this article, I will mainly discuss how to avoid reference cycle in ARC.

<br/>

## Prerequisite

Before we start, we should know the difference between strong / weak reference during the memory management.

### 1. ARC (Auto Reference Counting)

ARC works by counting the number of references to each object. When the reference count is larger than 0, then the object will keep alive. However, when the reference count drops to 0, then the object is definitely unreachable, and will be deallocated.

### 2. Strong reference

If there is a strong reference to the object, then the reference count of the object will increase by 1. In other words, as long as there are strong references to the object, then the reference count of the object will be larger than 0, and it will keep alive. However, if there are no strong references to the object, then the reference count will drop to 0, and then it will be recycled by ARC.

### 3. Weak reference

However, the weak reference does not increase the reference count of the object. It is noteworthy that, a weak reference is set to **nil** when there are no strong references to the object.

<br/>

## Reference cycle

Reference cycle happens when there are strong references pointing to each other, which causes the object will never be deallocated.

Let's have a look at the following examples below, in order to understand the reference cycle problem.

First of all, I have created a class "MZObject", in which I have declared a property with **strong reference** to the block called "referenceCycleCompletionBlock":

```c
#import "MZObject.h"

typedef void(^MZReferenceCycleCompletionBlock)(void);

@interface MZObject ()

// "MZObject" has a strong reference to the block
@property (nonatomic, strong) MZReferenceCycleCompletionBlock referenceCycleCompletionBlock;

@property (nonatomic, strong) NSString *text;

@end

@implementation MZObject

...

@end
```

Then, I have added some logs to print the initialization and dealloc of the "MZObject":

```c
#pragma mark - Initialize

- (instancetype)init {
	self = [super init];
	if (self) {
		NSLog(@">>>>>>>>>> Initialize");
	}
	return self;
}

#pragma mark - Dealloc

- (void)dealloc {
	NSLog(@">>>>>>>>>> Dealloc");
}
```

#### Scenario 1:

The following code will create a reference cycle.

Because self ("MZObject") already has a strong reference to the block, while the block has a strong reference to self ("MZObject"). So they have strong reference pointing to each other.

![Reference cycle](/assets/img/2019-02-20-reference-cycle/reference-cycle.png)

In this case, "MZObject" will never be deallocated.

```c
- (void)testReferenceCycle1 {
	// This will create reference cycle
	self.referenceCycleCompletionBlock = ^{
		NSLog(@">>>>>>>>>> Test reference cycle");
		self.text = @"text";
		NSLog(@">>>>>>>>>> Text: %@", self.text);
	};

	self.referenceCycleCompletionBlock();
}
```

If we run the project, we can see ">>>>>>>>>> Dealloc" is never printed out in the console.

```
2019-02-20 16:28:30.668641+0800 ReferenceCycleDemo[75335:15251817] >>>>>>>>>> Initialize
2019-02-20 16:28:30.668798+0800 ReferenceCycleDemo[75335:15251817] >>>>>>>>>> Test reference cycle
2019-02-20 16:28:30.668891+0800 ReferenceCycleDemo[75335:15251817] >>>>>>>>>> Text: text
```

#### Scenario 2:

We can fix the issue in Scenario 1 by adding "weakSelf" and "strongSelf".

In this case, "weakSelf" only has a weak reference to "MZObject", therefore "MZObject" can be deallocated successfully.

![Reference cycle](/assets/img/2019-02-20-reference-cycle/fix-reference-cycle.png)

```c
- (void)testReferenceCycle2 {
	// This will NOT create reference cycle
	__weak typeof(self) weakSelf = self;
	self.referenceCycleCompletionBlock = ^{
		typeof(self) strongSelf = weakSelf;
		NSLog(@">>>>>>>>>> Test reference cycle");
		strongSelf.text = @"text";
		NSLog(@">>>>>>>>>> Text: %@", strongSelf.text);
	};

	self.referenceCycleCompletionBlock();
}
```

If we run the project, we can see ">>>>>>>>>> Dealloc" is successfully printed out in the console.

```
2019-02-20 16:31:28.940376+0800 ReferenceCycleDemo[75386:15255699] >>>>>>>>>> Initialize
2019-02-20 16:31:28.940579+0800 ReferenceCycleDemo[75386:15255699] >>>>>>>>>> Test reference cycle
2019-02-20 16:31:28.941839+0800 ReferenceCycleDemo[75386:15255699] >>>>>>>>>> Text: text
2019-02-20 16:31:28.942006+0800 ReferenceCycleDemo[75386:15255699] >>>>>>>>>> Dealloc
```

#### Scenario 3:

Then, you may wonder, why do we need another "strongSelf"?

Let's have a try with only "weakSelf", while we create a background thread and let it sleep for 10 seconds, before we make use of "weakSelf".

```c
- (void)testReferenceCycle3 {
	// This will NOT create reference cycle
	__weak typeof(self) weakSelf = self;
	self.referenceCycleCompletionBlock = ^{
		dispatch_queue_t serialQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
		dispatch_async(serialQueue, ^{
			sleep(10);
			NSLog(@">>>>>>>>>> Test reference cycle");
			weakSelf.text = @"text";

			// The text will be "nil",
			// Because "self" is deallocated
			// And "weakSelf" is pointing to "nil"
			NSLog(@">>>>>>>>>> Text: %@", weakSelf.text);
		});
	};

	self.referenceCycleCompletionBlock();
}
```

If we run the project, we can see ">>>>>>>>>> Text: (null)" is printed out in the console.

```
2019-02-20 16:38:58.918655+0800 ReferenceCycleDemo[75463:15263686] >>>>>>>>>> Initialize
2019-02-20 16:38:58.918836+0800 ReferenceCycleDemo[75463:15263686] >>>>>>>>>> Dealloc
2019-02-20 16:39:08.924087+0800 ReferenceCycleDemo[75463:15263740] >>>>>>>>>> Test reference cycle
2019-02-20 16:39:08.924357+0800 ReferenceCycleDemo[75463:15263740] >>>>>>>>>> Text: (null)
```

The reason is that, by the time we execute the code, "MZObject" has already been deallocated. Then "weakSelf" will be pointing to nil. Hence, "weakSelf.text = xxx;" is not being executed properly.

#### Scenario 4:

Hence, we can fix the issue in Scenario 3 with a "strongSelf":

```c
- (void)testReferenceCycle4 {
	// This will NOT create reference cycle
	__weak typeof(self) weakSelf = self;
	self.referenceCycleCompletionBlock = ^{
		typeof(self) strongSelf = weakSelf;
		dispatch_queue_t serialQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
		dispatch_async(serialQueue, ^{
			sleep(10);
			NSLog(@">>>>>>>>>> Test reference cycle");
			strongSelf.text = @"text";
			NSLog(@">>>>>>>>>> Text: %@", strongSelf.text); // The text will NOT be "nil"
		});
	};

	self.referenceCycleCompletionBlock();
}
```

If we run the project, we can see ">>>>>>>>>> Text: text" is successfully printed out in the console.

```
2019-02-20 16:48:05.492885+0800 ReferenceCycleDemo[75624:15272848] >>>>>>>>>> Initialize
2019-02-20 16:48:15.496200+0800 ReferenceCycleDemo[75624:15272903] >>>>>>>>>> Test reference cycle
2019-02-20 16:48:15.496517+0800 ReferenceCycleDemo[75624:15272903] >>>>>>>>>> Text: text
2019-02-20 16:48:15.496715+0800 ReferenceCycleDemo[75624:15272903] >>>>>>>>>> Dealloc
```

#### Scenario 5:

However, it does not mean we should use "weakSelf" and "strongSelf" in all the blocks. **"weakSelf" and "strongSelf" are only necessary when the instance of the class has a strong reference to the block.**

For example, if the block is declared inside a function, then it will not cause any reference cycle problem. Because it is equivalent to declare the block as a **local variable** inside a function (See: Scenario 6), and therefore it will be deallocated after the function is executed.

```c
- (void)testReferenceCycle5 {
	// This will NOT create reference cycle
	[self performBlockWithCompletion:^{
		NSLog(@">>>>>>>>>> Test reference cycle");
		self.text = @"text";
		NSLog(@">>>>>>>>>> Text: %@", self.text);
	}];
}
```

If we run the project, we can see ">>>>>>>>>> Dealloc" is successfully printed out in the console.

```
2019-02-20 16:57:08.729539+0800 ReferenceCycleDemo[75732:15282020] >>>>>>>>>> Initialize
2019-02-20 16:57:08.729704+0800 ReferenceCycleDemo[75732:15282020] >>>>>>>>>> Test reference cycle
2019-02-20 16:57:08.729799+0800 ReferenceCycleDemo[75732:15282020] >>>>>>>>>> Text: text
2019-02-20 16:57:08.729911+0800 ReferenceCycleDemo[75732:15282020] >>>>>>>>>> Dealloc
```

#### Scenario 6:

Scenario 5 can be further re-write with a **local variable** in Scenario 6, and both scenario 5 and scenario 6 are equivalent. Hence, the reference cycle problem will never occur.

```
- (void)testReferenceCycle6 {
	// This will NOT create reference cycle
	void (^completionBlock)(void) = ^{
		NSLog(@">>>>>>>>>> Test reference cycle");
		self.text = @"text";
		NSLog(@">>>>>>>>>> Text: %@", self.text);
	};

	[self performBlockWithCompletion:completionBlock];
}
```

If we run the project, we can see ">>>>>>>>>> Dealloc" is successfully printed out in the console.

```
2019-02-20 16:59:03.265632+0800 ReferenceCycleDemo[75757:15284207] >>>>>>>>>> Initialize
2019-02-20 16:59:03.265829+0800 ReferenceCycleDemo[75757:15284207] >>>>>>>>>> Test reference cycle
2019-02-20 16:59:03.265951+0800 ReferenceCycleDemo[75757:15284207] >>>>>>>>>> Text: text
2019-02-20 16:59:03.266052+0800 ReferenceCycleDemo[75757:15284207] >>>>>>>>>> Dealloc
```

<br/>

## Conclusion

This article mainly discribes how to avoid reference cycle in ARC. Only if we pay more attention to the reference cycle problems, can we improve the memory usage and avoid any memory leak issues.

<br/>

## Download

The example code can be downloaded in the following [link]({{ site.url }}/assets/file/2019-02-20-reference-cycle/ReferenceCycleDemo.zip)