---
title: Literal snapshot testing
date: "2024-05-05"
---

Snapshot testing is a type of so-called "golden master" testing, where an assertion is made that a value matches some previously captured output. [Insta](https://insta.rs/) is a library providing snapshot test tooling in the Rust ecosystem, and their [description of snapshot testing](https://insta.rs/#what-does-this-look-like) is a useful introduction.

## Snapshot testing is most often done on text

Even when used for testing UI components, as commonly done in the [React ecosystem using Jest](https://jestjs.io/docs/snapshot-testing), snapshot testing is most often done on text. In React, the snapshots are made up of the component as rendered into HTML, rather than a visual representation. Presumably this is because browsers may not visually render the same HTML/CSS with pixel perfect consistency, so it is more stable to test the HTML output directly.

## Snapshot testing an embedded systems user interface.. with images

I've been working on a [firmware for the PineTime smartwatch](https://github.com/JoshMcguigan/mesozoic), and wanted a way to develop automated tests of the user interface. To scratch this itch I developed some barebones test infrastructure supporting snapshot testing, using PNGs captured by running the user interface code inside a simulator.

* [test infrastructure](https://github.com/JoshMcguigan/mesozoic/blob/52363b79e55163409d659ec8c74ed645a03d759a/app/src/test_infra.rs)
* [example of the test code](https://github.com/JoshMcguigan/mesozoic/blob/52363b79e55163409d659ec8c74ed645a03d759a/app/src/app.rs#L278)
* ["golden record" snapshot](https://github.com/JoshMcguigan/mesozoic/blob/52363b79e55163409d659ec8c74ed645a03d759a/app/snapshots/mesozoic_app%3A%3Aapp%3A%3Atests%3A%3Ainit_and_paired.golden.png)

The test code first uses a simulator to render the user interface into a PNG. It then checks for an existing snapshot. If there is one, it asserts that the current PNG matches the snapshot. If the developer is purposefully making a UI change, they can compare the new and old snapshots to ensure the change matches their expectations. If a snapshot doesn't exist yet, the current PNG is saved as the golden record. If the developer is adding a new test, they should manually validate that this snapshot matches the intended UI.

## Snapshot diffs in pull requests

Snapshots are saved in the git repository. There can be downsides to including large files in git, but these PNG files are actually smaller than most text files in the repo. And although git itself doesn't natively know how to nicely diff images, both github and gitlab do provide nice tooling for diff'ing changes to an image. You can see an example in this [pull request modifying the battery state UI](https://github.com/JoshMcguigan/mesozoic/pull/2/files#diff-2a94fc494a26e2b5b3d14ceaf2c07a125646cfc58e5f22e6cc0792a246f3d0e6).

## Paving the road ahead

While this snapshot tooling is useful in its current state, there is a lot of missing polish:

* Snapshots updates are done by manually deleting the golden record to force a new one to be recorded
* The current assertion is done using the standard `assert_eq` macro, which prints out all the bytes two PNGs when a test fails
* When a test fails, comparing snapshots is a manual process with no tooling built around it to simplify the process
* There is some clunkiness dealing with picking a name for the saved golden record file
  * Insta [also deals with this](https://internals.rust-lang.org/t/discovering-the-current-test-name/15325), although they seem to have found a solution which has a nicer developer experience than what I settled for

There are probably lots more things missing here that I haven't run into yet. That said, even in its current (very simplistic) state, I find these tests improve my confidence when making changes to UI code.