+++
title = "My experience using mutation testing in production"
date = "2025-01-19"

[taxonomies]
tags = ["mutation testing", "programming", "testing"]
+++
### What is mutation testing?

Mutation testing is a technique for improving the correctness of your software, answering the question: do my unit tests actually cover every branch of my code? Mutation testing tools are external programs that make syntactically-legal alterations to your codebase, run your test suite (which is left unaltered), and check whether any of your tests fail. Effectively, it's like deliberately adding bugs to your code in a systematic way and seeing if you're already testing for those bugs (and, of course, reverting the bugs after testing is complete). For example, take this toy function:

```python
def filter_records(records):
    for record in records:
        if record.quality < 30:
            continue
        yield record
```

The mutation testing library will flip the `<` to `>`, `yield record` to `yield None`, `30` to a bunch of different numbers, and so forth. It's mostly limited to operators, strings, and constants, and the details vary by tool and language. Only one of these so-called mutants is tested at a time - while it would be possible to create multiple mutants for a single test run, practical experience shows that this doesn't add much value, and the run time would explode exponentially.

If none of your tests fail even after adding these "bugs", it shows that your tests are incomplete - adding bugs should cause failing tests! Your job is then to either write more unit tests or refactor such that the things being mutated cease to exist. This is complementary to code coverage tools, which can only show whether a line was executed during the run of a test suite. It's still possible - likely even - that you can have 100% line coverage while still missing some behavior. Take this example:

```python
if x and y:
    launch_rocket()
if z:
    load_cargo_onto_rocket()
```

Suppose you have one test where `x` and `y` evaluate to `True` but `z` is `False`, and another test where `x` or `y` is `False` and `z` is True. In one test, `launch_rocket()` will run, in the other, `load_cargo_onto_rocket()` will run, but you still haven't exercised the scenario where `x`, `y`, and `z` are all `True`, which would reveal that this code is going to launch an empty rocket and then try to load cargo into a vehicle that is no longer there. A test coverage tool will correctly inform you that all four lines are tested, but the most critical behavior is ignored.

### My experience with mutmut

I had a greenfield project at work and I decided it was a great opportunity to give mutation testing a whirl. This was for a Python application that, in broad terms, took raw sequencing data and determined the error rate of an enzyme. I looked at [Cosmic Ray](https://github.com/sixty-north/cosmic-ray), [Mutatest](https://github.com/EvanKepner/mutatest), and [mutmut](https://github.com/boxed/mutmut). I ended up choosing mutmut as it just seemed the most polished at the time. I never did any rigorous comparison so I don't want to endorse it over the others, but I was mostly pleased with it.

Overall, I'm super happy with mutation testing, but not because it caught many bugs. In fact, I think it really only found 1 or 2 true positives, and they were relatively minor. The real benefit was that it deeply impacted the design of the codebase such that it was the most testable piece of software I've ever written. This happened because each time I added some new feature, I had to immediately consider whether I wanted to write dozens of unit tests to eliminate the mutants, or whether I wanted to refactor in a way that made it easier to test. Having a bunch of new mutants show up is often a sign of unnecessary complexity.

Towards the end of the project, I had been thinking that it was probably not worth it to test the main entrypoint function in the tool as I'd need to essentially simulate an entire run of the application starting from raw sequencing data, but as I was so close I decided to spend a few hours writing it, and I'm glad I did. Having the entire codebase killing all its mutants not only gave me confidence that the code was correct, I could also fearlessly make changes.

### mutmut has a few flaws

Some mutation testing tools do everything without altering the source on disk - the code is loaded, altered and tested in memory (which is apparently possible!). Because each mutant is independent of every other, this is embarrassingly parallelizable.  mutmut, on the other hand, writes each change to disk and then runs the test suite, one mutant at a time. This is certainly a much simpler design, but in addition to being slower, if you cancel the run partway through, the mutant that was being tested at the time will persist on disk! This isn't an issue if you committed your source just before the run since it would make reverting it trivial, but I typically want to see if my tests pass before I commit, so I had to sift through all of the deliberate changes I had made in order to find a single-character alteration.

### Advice on adopting mutation testing

1. **Fast test suites are essential**

If you can run your entire test suite in one second, and you have 300 mutants to test, then adding mutmut to your workflow means it now takes five minutes to run your tests. There are a couple things you can do to optimize this, fortunately. 

First, the key is to observe that the vast majority of the time, mutants will in fact cause tests to fail, so you want to optimize for failing fast. If you have any property-based tests or slow-running tests in general, you can configure pytest to run them last by marking them as slow. Often, mutants will be killed by several different tests, so if you can kill them with fast tests, you can shave off meaningful amounts of time.

Second, while a bit obvious, is to not run mutmut until you're ready to commit, or only run in CI. Since it will identify a number of false positives in any new code, any time spent resolving those is wasted if you end up changing your design. I found that just running unit tests while developing, and then only running property-based and mutation tests once I thought I had something worthwhile ended up being a good compromise. The iteration time on finding out if I had architectural flaws was still fast enough that I never had to do any major refactors.

2. **Start early**

After my initial success, I tried bolting on mutation testing to an existing project and it was a nightmare - there were hundreds of surviving mutants, and resolving them would require several refactors and just tons and tons of work that I simply couldn't justify. Having the tool constrain the design from the start really is critical. It's not impossible to adopt it later, but it does require a non-trivial investment.

3. **Skip plotting functions and other "untestable" code, but here there be dragons**

mutmut works on an opt-in basis, so it will only modify files you explicitly tell it to. My tool generated a bunch of figures with matplotlib, which is not practically testable, and for those functions I just kept them in a separate module. I do have the habit of doing some light data wrangling in such code (e.g. something like getting all the values from a dictionary and plotting a histogram, instead of just passing in a list of values directly). Although it's trivial, "this can't possibly be wrong" is the thing that everyone writing a bug is telling themselves, so I tried to move as much of that code as possible to the modules where mutmut could evaluate them.

4. **Mutation testing tools are still somewhat limited**

Only being able to modify operators, strings, and constants is powerful but doesn't provide complete coverage. Notably lacking from all of the tools I looked at is the ability to alter method calls. For example, they can't remove `.strip()` from a string variable, or swap `.is_upper()` with `.is_lower()`. To do so would require type inference, and the libraries for doing that in Python don't seem like they could easily integrate with a mutation testing tool, if they would even work at all. I have high hopes for [Ruff](https://github.com/astral-sh/ruff), which is implementing a [type inference engine](https://github.com/astral-sh/ruff/issues?q=state%3Aopen%20label%3A%22red-knot%22), and once that matures I think there would be a great opportunity to merge that into existing tools or to design one around it. Until then, it's still a great technique, but it's important to recognize this limitation.

5. **Mutation testing won't catch all bugs**

While the combination of unit tests, property-based tests and mutation tests did basically result in functions that were all correct (I mean, as far as I could tell), I still encountered a bug that was more strategic in error. In essence, I had told my tool to do the Wrong Thing, and then had tests that ensured the Wrong Thing was being done exactly as instructed. This kind of error will never be caught by any of these techniques, so we'll always need a human or human-level intelligence to review the code and think critically about it.
