---
title: Combine Results for Improved Rust Validation Logic
date: "2019-02-23"
---

The error handling features within Rust are some of my favorite things about the language. The `Result` and `Option` types save developers from using implicit placeholder values (things like `-1` and `null` respectively) in almost all cases. Additionally, the `try`/`?` operators make it ergonomic to handle these error conditions, while the compiler ensures you can't use the underlying value without first confirming it is ok.

This system works great when you are in a function which returns a `Result` and you want to exit at the first error you come to. However, it can be challenging if your goal is to try a few failure-prone things and return each of the errors, rather than just the first error. This is the problem [multi_try](https://github.com/JoshMcguigan/multi_try) attempts to solve. 

## A Simple Validation Function

Throughout this blog post we'll use the problem of validating an email to demonstrate the various approaches. Our goal will be to convert from the `Email` struct to the `ValidatedEmail` struct, and return `EmailValidationErr` if there are any problems.

```rust
struct Email<'a> {
    to: &'a str,
    from: &'a str,
    subject: &'a str,
    body: &'a str,
}

struct ValidatedEmail<'a> {
    to: &'a str,
    from: &'a str,
    subject: &'a str,
    body: &'a str,
}

enum EmailValidationErr {
    InvalidEmailAddress,
    InvalidRecipientEmailAddress,
    InvalidSenderEmailAddress,
    InvalidSubject,
    InvalidBody,
}
```

The function below demonstrates a typical approach to performing this type of validation which will validate the `to`, `from`, `subject`, and `body` fields in that order, and return either the `ValidatedEmail` or the first `EmailValidationErr`.

```rust
fn validate_email(email: Email) -> Result<ValidatedEmail, EmailValidationErr> {
    Ok(
        ValidatedEmail {
            to: validate_address(email.to)
                    .map_err(|_| EmailValidationErr::InvalidRecipientEmailAddress)?,
            from: validate_address(email.from)
                    .map_err(|_| EmailValidationErr::InvalidSenderEmailAddress)?,
            subject: validate_subject(email.subject)?,
            body: validate_body(email.body)?,
        }
    )
}
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=178e9f4a6a3f4351f7b24f7fae113f3f)

## Returning All Errors 

If these error messages are being returned to a user, it would be nice if we could provide a message about all of the validation errors, rather than just the first error. A potential approach for this is demonstrated below.

```rust
fn validate_email(email: Email) -> Result<ValidatedEmail, Vec<EmailValidationErr>> {
    let mut errors = vec![];
    let to = validate_address(email.to)
        .unwrap_or_else(|_e| {
            errors.push(EmailValidationErr::InvalidRecipientEmailAddress);
            ""
        });
    let from = validate_address(email.from)
        .unwrap_or_else(|_e| {
            errors.push(EmailValidationErr::InvalidSenderEmailAddress);
            ""
        });
    let subject = validate_subject(email.subject)
        .unwrap_or_else(|e| {
            errors.push(e);
            ""
        });
    let body = validate_body(email.body)
        .unwrap_or_else(|e| {
            errors.push(e);
            ""
        });

    if !errors.is_empty() {
        return Err(errors);
    }

    Ok( ValidatedEmail { to, from, subject, body } )
}
```

Note how the error type in the return value changed from a `EmailValidationErr` to a `Vec<EmailValidationErr>`, indicating we are now returning all of the validation errors rather than just the first. This could provide a nice UX benefit, but we pay the price in code complexity. 

Most critically we are giving up on an important guarantee that idiomatic Rust typically provides us. In order to continue past the first error, we use `unwrap_or_else` to provide a placeholder value to the fields of our email struct (in this case we use the empty string), and then we push errors into the error vec. The downside to this approach is that once we initialize the fields to an empty string, the compiler no longer knows/cares that they are invalid, so it cannot enforce that we check for errors before using the values (this code would still compile if I removed the `if !errors.is_empty()` block). 

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=463daedba471031c98e7237ee00d91d8)

## Introducing multi_try

What we really want is to combine the compiler guarantees of the first approach, with the UX benefits of the second approach, and that is where [multi_try](https://github.com/JoshMcguigan/multi_try) comes in. 

```rust
use multi_try::MultiTry;

fn validate_email(email: Email) -> Result<ValidatedEmail, Vec<EmailValidationErr>> {
    let (to, from, subject, body) = validate_address(email.to).map_err(|_| {
        EmailValidationErr::InvalidRecipientEmailAddress
    }).and_try(validate_address(email.from).map_err(|_| {
        EmailValidationErr::InvalidSenderEmailAddress
    })).and_try(
        validate_subject(email.subject)
    ).and_try(
        validate_body(email.body)
    )?;

    Ok(ValidatedEmail { to, from, subject, body })
}
```

This approach provides the best of both worlds. We are still returning a `Vec<EmailValidationErr>` to get the UX benefit of returning all of the errors rather than just the first, and the compiler ensures we check for errors before using the `to`, `from`, `subject` and `body` fields to bulid the `ValidatedEmail`. Unlike our second attempt, if I removed the error handling (perhaps by removing the `?`) this code would no longer compile.

## Feedback Wanted

This crate is in the experimental phase, and all feedback is appreciated. Feel free to create an [issue](https://github.com/JoshMcguigan/multi_try/issues) to express any suggestions, questions, or criticisms. 

#### Community Involvement

I've edited this blog post based on the changes to `multi_try` from Github user [sunjay](https://github.com/sunjay). Sunjay submitted a very well written [pull request](https://github.com/JoshMcguigan/multi_try/pull/1) improving the `multi_try` API by implementing it as an extension trait on the standard `Result` type.
