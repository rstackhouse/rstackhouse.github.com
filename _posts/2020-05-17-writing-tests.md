---
layout: post
title: Editing Tests
category: posts
---

Software developers are human beings. Human beings are fallible. Meaning we hardly ever get anything right on the first pass. So why do we call it "writing code" instead of "editing code"? Why does the English language need two words to describe two activities that are inexorably linked? You can't have [published] writing without editing, and you can't have editing without writing. Professional authors can tell you which is more important. Philosophy aside; a colleague recently asked me, "What consitutes a good test?" I interpreted his question to mean, "How dow I write good tests?" The answer: it depends.

That question combined with a desire to dig into the [Fluent Assertions](https://fluentassertions.com/) library led me to this post: https://medium.com/@leandronherenu/unit-testing-its-importance-and-how-to-do-it-using-xunit-moq-and-fluentassertions-c-d8a4402117cc. Mr. Hereñu did a great job putting together that article. It definitely fit the bill for "How to use Fluent Assertions with xUnit?" However, I got to the bottom of the article, and I saw that the test was asserting more than one thing. This is not to say that the test was invalid. It is just that people more [qualified](http://programmaticallyspeaking.com/one-assertion-per-test-please.html) [than I](https://mfranc.com/unit-testing/good-unit-test-one-assert/) have suggested that "one assertion per test" is a prefered pattern. People [disagree about this](https://softwareengineering.stackexchange.com/questions/7823/is-it-ok-to-have-multiple-asserts-in-a-single-unit-test).  Mr. Jain points out that it is [maybe not the best name](https://blogs.agilefaqs.com/2010/11/14/single-assert-per-unit-test-myth/). Oh well. Software developers tend to be bad at naming things. Maybe ["One reason to fail"](https://lostechies.com/jimmybogard/2008/10/14/acceptable-test-failures/) is better?

At any rate, I forked Mr. Hereñu's repo, and pretended as though we were on the same team. The first change I made was just to update to .NET Core 3.1. No judgment there. It is just what I am set up for. 


The second revision I made was to simplify the test.

```
        [Fact]
        public void Should_NotSendLetter_When_SenderIsNotOlderEnough()
        {
            // Arrange 
            var receiver = new Person();

            // Don't actually care about the age of the person as the age validator
            //   is not under test and it is mocked
            var letters = new List<Letter> { new Letter { Sender = new Person {}, Receivers = new List<Person>{ receiver } } }; 

            this.ageValidator
                .Setup(x => x.IsNotOlderEnough(It.IsAny<Person>())) // No need for an exact match. Don't over constrain.
                .Returns(false);

            // Act
            this.lettersDispatcher.Dispatch(letters);

            // Assert
            receiver.ReceivedLetters.Should().HaveCount(0);
        }
```

As you can see, I am only asserting one thing: the receiver shouldn't have any mail. Again, one assertion per test is a matter of opinion. To me, it increases readability. It also increases granularity. One assertion per test is the logical application of the "S" in SOLID priniciples to unit testing. A test should have only one reason to exist. In my view, testing code this way pushes you toward making one change at a time. Which seems to be a fairly widely accepted debugging practice. Since the test depends on the return value of the mocked age validator, I am not putting any real effort into the input into the mock. I also removed the matching constraint on the setup for the mocked age validator as it doesn't serve any real utility, at least that I could find, in the test. I just need to know, when the age validator returns false, no mail is sent.

Next, I simply added a test for the language validator. Not much to see here.

```
        [Fact]
        public void Should_NotSendLetter_When_TheBodyOfItHasBadWords()
        {
            // Arrange 
            var receiver = new Person();

            // Don't actually care about the age of the person as the age validator
            //   is not under test and it is mocked
            var letters = new List<Letter> { new Letter { Sender = new Person {}, Receivers = new List<Person>{ receiver } } }; 

            this.ageValidator
                .Setup(x => x.IsNotOlderEnough(It.IsAny<Person>())) // No need for an exact match. Don't over constrain.
                .Returns(true);

            this.badWordsValidator
                .Setup(x => x.ThereAreNotBadWords(It.IsAny<string>())) // No need for an exact match. Don't over constrain.
                .Returns(false); 

            // Act
            this.lettersDispatcher.Dispatch(letters);

            // Assert
            receiver.ReceivedLetters.Should().HaveCount(0);
        }
```

Then I added a test for the happy path. Typically, I do this first, but that's just me.

```
        [Fact]
        public void Should_SendLetterToReceivers_When_AllValidationsPass()
        {
            // Arrange 
            var receiver = new Person();

            var letters = new List<Letter> { new Letter { Sender = new Person {}, Receivers = new List<Person>{ receiver }, Body = "Some Message"  } }; 

            this.ageValidator
                .Setup(x => x.IsNotOlderEnough(It.IsAny<Person>())) // No need for an exact match. Don't over constrain.
                .Returns(false);

            this.badWordsValidator
                .Setup(x => x.ThereAreNotBadWords(It.IsAny<string>())) // No need for an exact match. Don't over constrain.
                .Returns(true); 

            // Act
            this.lettersDispatcher.Dispatch(letters);

            // Assert
            receiver.ReceivedLetters.Should().HaveCount(1);
        }
```

At this point, I have three occurances of setting up the age validator. So I refactored out this method.

```
        protected void SetupAgeValidator(bool senderIsMinor)
        {
            this.ageValidator
                .Setup(x => x.IsNotOlderEnough(It.IsAny<Person>())) // No need for an exact match. Don't over constrain.
                .Returns(!senderIsMinor);
        }
```

Which contains a logic error `!senderIsMinor` that I didn't catch until later. It should have been `senderIsMinor`, which brings me to some of my issues with the `!` operator, but that is a different topic altogether.

I replaced

```
this.ageValidator
                .Setup(x => x.IsNotOlderEnough(It.IsAny<Person>())) // No need for an exact match. Don't over constrain.
                .Returns(/* a bool here */);
```

With either `SetupAgeValidator(senderIsMinor:true)` or `SetupAgeValidator(senderIsMinor:false)`

Next I implemented the test to ensure a minor's family is notified if they try to send a message.

```
        [Fact]
        public void Should_NotifyToTheSenderFamily_When_SenderIsNotOlderEnough()
        {
            // Arrange 
            var receiver = new Person();

            var momName = "Liz";

            var mom = new Person{ Name=momName };

            var family = new List<Person>{mom};

            var letters = new List<Letter> { new Letter { Sender = new Person { Relatives = family }, Receivers = new List<Person>{ receiver }, Body = "Some Message" } }; 

            SetupAgeValidator(senderIsMinor: true);

            // Act
            this.lettersDispatcher.Dispatch(letters);

            // Assert
            // NOTE: This is kind of a slippery slope. This test uses reference equality.
            notificationSender.Verify(n => n.Send(It.IsAny<string>(), family), Times.AtLeastOnce());
        }
```

The line `notificationSender.Verify(n => n.Send(It.IsAny<string>(), family), Times.AtLeastOnce());` has a potential to create false negatives in the future. What if `family` contains siblings in the future? At that point, logically, we would only want to notify the parents. If this were going to be production code, I might express this test differently or override `Equals` in the `Person` class.

Then, I added a test for not sending empty messages.

```
[Fact]
        public void Should_SendLetterToReceivers_When_ThereIsNoMessage()
        {
            // Arrange 
            var receiver = new Person();

            var letters = new List<Letter> { new Letter { Sender = new Person {}, Receivers = new List<Person>{ receiver } } }; 

            SetupAgeValidator(senderIsMinor:false);

            this.badWordsValidator
                .Setup(x => x.ThereAreNotBadWords(It.IsAny<string>())) // No need for an exact match. Don't over constrain.
                .Returns(true); 

            // Act
            this.lettersDispatcher.Dispatch(letters);

            // Assert
            receiver.ReceivedLetters.Should().HaveCount(0);
        }
```

This method should have been called `Should_SendNotLetterToReceivers_When_ThereIsNoMessage`. This is why we have code reviews. I followed that with a test for not sending messages with an empty subject.

Eventually, I wound up with the following refinements to the unit test suite.

```
        protected void SetupAgeValidator(bool senderIsMinor)
        {
            this.ageValidator
                .Setup(x => x.IsNotOlderEnough(It.IsAny<Person>())) // No need for an exact match. Don't over constrain.
                .Returns(senderIsMinor);
        }

        protected void SetupLanguageValidator(bool cleanSpeach)
        {
            this.badWordsValidator
                .Setup(x => x.ThereAreNotBadWords(It.IsAny<string>())) // No need for an exact match. Don't over constrain.
                .Returns(cleanSpeach); 
        }

        protected void BadWords () { SetupLanguageValidator(cleanSpeach:false); }

        protected void NoBadWords () { SetupLanguageValidator(cleanSpeach:true); }

        protected void MinorSender () { SetupAgeValidator(senderIsMinor:true);  }

        protected void AdultSender () { SetupAgeValidator(senderIsMinor:false); }
```

Please note that the above refinements only hold any value _if there are multiple tests_. Maintained well, unit test suites get better with age. A "crappy" first test is not a bad effort. It is a starting place. Unit tests have no intrinsic value (read we don't get paid to write them). Their value comes from allowing us to add value to production code with confidence. Easily understandable (one reason to fail) tests theorectically allow us to make small changes faster.

Context is king. If the test from Mr. Hereñu's article was to be the only test ever to be written for this code, then none of my changes would have provided any value. Also, as always, do what works for your team. 
