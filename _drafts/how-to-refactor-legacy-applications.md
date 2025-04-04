---
layout: post
title: "Lets take a chunk from that"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang refactoring
---

Explain how refactoring legacy applications can work if you take one piece out at a time.
Example message-broker.
Things that don't change will be left as they are.
Things that change frequently will be refactored and extracted (either into package or their own repo)

Structure:
1. Why do we refactor?
    - makes the code easier to understand
    - make it easier to update/change existing functionality
    - make it easier to integrate new features
    - make debugging and bug-hunting easier
    - all of that reduces the maintenance and development cost in the long run
    - its like reorganising your toolbox. You invest a bit of time now to be faster and more efficient later on.
2. Why dont we refactor?
    - we don't refactor to increase performance (but it could be a positive side effect)
    - we don't refactor to solve bugs (this should be done in a bug-ticket outside of the refactoring process)
3. Problems with refactoring big legacy projects
    - Test-Coverage is most of the time really bad
    - Unknown/Magic dependencies
    - a lot of dependencies between project packages
    - risk of breaking existing features is quite high
    - non-modular architecture makes it hard to contain changes
4. How to refactor? Like you would eat an elephant. Piece by Piece
    - To determine where to start with the refactoring IMO there is one important metric: How often does this code change?
        - Was the code not changed for a long time? Then what benefit are you expecting from changing it now? If there is no real benefit then the risk of changing is always winning and you should leave it.
            ->  exceptions could be if you are changing some underlying project structure and you need to change some packages to fit that
            ->  another exception could be if the code is really complicated to debug and needs to be investigated when debugging a lot. 
                An example would be if you have a webserver which contains a function that maps routes to handlers and whenever you debug you have to go through that function.
        - Is the code frequently changed? If so can we expect an improvement by refactoring it? Does it outweight the risk that we break something?
    - If we have found the part of the code that we want to refactor we need to ask ourself if we can break it down into smaller tasks. If you can break it down you need to reevaluate every task again if you want to refactor it.
      if it does not make sense to split the refactoring into smaller chunks we can start. From my experience it is better to finish some very small refactorings then start a big one that will never be finished.
      We need to keep in mind that after we refactor the change most likely needs to be reviewed, tested and approved and while that is happening the regular development continues.
      The bigger our refactoring is and the longer our review-process takes, the higher the chance that there are conflicts when its time to merge and that we need to update the refactoring and start the process all over again.
      So IMO every task of refactoring should have a developing-time of less than one day. Everything above that has caused a lot of problems.
5. But how do I start? With Tests!
    - The biggest problem with refactorings is that they often break existing functionality. To keep the risk of breaking something small I suggest you start with adding some tests that test happy and unhappy paths.
    - The more time we invest now into writing tests for the current behaviour the faster the review+test process will be and the smaller the risk of the refactoring actually becomes
6. Can we now start with refactoring?
   Yes! We can finally start with refactoring. If possible I would recommend to focus on two things:
    1. Move the functionality into an isolated package. This package should have few to no static dependencies (use interfaces and dependency-injection for those) and follow the single-responsibility-principle.
       Make sure you have the new package covered with unittests. This can then be injected into the existing code using dependency injection.
       This sometimes means that you are sometimes left with a nearly empty function that only take the dependency, calls it and does some transformation.
       This completely fine and you should only try to remove the function and move the changes further up the hierarchy if the changes are super small. If not create a followup-ticket to look into this later. 
       The last thing we want to do now is to run into the trap of bloating the scope and then not finishing the ticket.
    2. Boring is good. Was there an abstraction-layer that could be replaces with a simple switch-case? Are there generic functions that then use a switch-case on the type? Remove them and replace them with plain and simple code.
       We dont want to write clever code we want to write code as boring as possible.
       "But what if we need that flexibility later?" in short, in 99% of the case you won't. So must remove it and if you actually need it THEN it is time to develop it. If the code is plain, boring and tested then adding flexibility later on will be simple.
