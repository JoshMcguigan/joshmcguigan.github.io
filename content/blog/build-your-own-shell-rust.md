---
title: Build Your Own Shell using Rust
date: "2018-11-17"
---

This is a tutorial on building your own shell using Rust, in the spirit of the [build-your-own-x](https://github.com/danistefanovic/build-your-own-x) list. Creating a shell is a great way to understand how the shell, terminal emulator, and OS work together.

## What is a shell?

A shell is a program which allows you to control your computer. It does this largely by making it easy to launch other applications. But a shell on it's own isn't an interactive application. 

Most users interact with the shell through a terminal emulator. A concise description of a terminal emulator by user [geirha](https://askubuntu.com/a/111149) follows:

> The terminal emulator (often just called terminal) is "just the window", yes. It runs a text based program, which by default is your login shell (which is bash in Ubuntu). When you type characters in the window, the terminal draws these characters in the window in addition to sending it to the shell's (or other program's) stdin. The characters the shell outputs to stdout and stderr get sent to the terminal, which in turn draws these characters in the window. 

In this tutorial, we'll write our own shell and run it inside our normal terminal emulator (wherever you'd typically `cargo run`). 

## A Starting Point

The simplest possible shell requires only a handful of lines of Rust code. Here we create a new string, which is used to hold the user input. The `stdin().read_line` function blocks until the user presses the enter key, then it writes the entire user input (including the newline from pressing enter) into our string. After stripping the newline character with `input.trim()` we attempt to run the command. 

```rust
fn main(){
    let mut input = String::new();
    stdin().read_line(&mut input).unwrap();

    // read_line leaves a trailing newline, which trim removes
    let command = input.trim(); 

    Command::new(command)
        .spawn()
        .unwrap();
}
```

After `cargo run`ing this, you should see a flashing cursor in your terminal which is waiting for input. Try typing `ls` and pressing `enter`, and you will see the `ls` command print the contents of the current directory, and then the shell will exit.

Note: These examples cannot be run in the [Rust Playground](https://play.rust-lang.org/) because the playground does not currently support stdin nor long running processes. 

## Accept Multiple Commands

We don't want our shell to exit after the user enters a single command. Supporting multiple commands is mostly a matter of wrapping the code above in a `loop`, and adding a call to `wait` on each child process to ensure we don't prompt the user for additional input before the current process finishes. I've also added a couple lines to print the `>` character in order to make it easier for the user to distinguish their input from the output of the processes they spawn. 

```rust
fn main(){
    loop {
        // use the `>` character as the prompt
        // need to explicitly flush this to ensure it prints before read_line
        print!("> ");
        stdout().flush();

        let mut input = String::new();
        stdin().read_line(&mut input).unwrap();

        let command = input.trim();

        let mut child = Command::new(command)
            .spawn()
            .unwrap();

        // don't accept another command until this one completes
        child.wait(); 
    }
}
```

Running this code you'll see that after running a first command, the prompt comes back so you can enter a second command. Try it out with, for example, the `ls` and `pwd` commands.

## Handling Args

If you try running the command `ls -a` on the shell above, it will crash. Since it is not aware of arguments, it tries to run a command called `ls -a`, but the proper behavior is running a command called `ls` with the argument `-a`. 

This is fixed below by splitting the user input on whitespace characters, and treating anything before the first whitespace as the name of the command (e.g. `ls`), while anything after the first whitespace is passed to that command as args (e.g. `-a`). 

```rust
fn main(){
    loop {
        print!("> ");
        stdout().flush();

        let mut input = String::new();
        stdin().read_line(&mut input).unwrap();

        // everything after the first whitespace character 
        //     is interpreted as args to the command
        let mut parts = input.trim().split_whitespace();
        let command = parts.next().unwrap();
        let args = parts;

        let mut child = Command::new(command)
            .args(args)
            .spawn()
            .unwrap();

        child.wait();
    }
}
```

## Shell Built-ins

It turns out there are certain commands that the shell cannot simply dispatch to another process. These are things which affect something internal to the shell, and thus must be implemented by the shell itself. 

Probably the most common example of this is the `cd` command. For an explanation of why `cd` must be a shell built-in, check out [this link](https://unix.stackexchange.com/a/38809). In addition to the shell built-in, there actually is a program called `cd`. The reasons for this duality is explained [here](https://unix.stackexchange.com/a/38819).

Below we add support to our shell for the `cd` built-in. 

```rust
fn main(){
    loop {
        print!("> ");
        stdout().flush();

        let mut input = String::new();
        stdin().read_line(&mut input).unwrap();

        let mut parts = input.trim().split_whitespace();
        let command = parts.next().unwrap();
        let args = parts;

        match command {
            "cd" => {
                // default to '/' as new directory if one was not provided
                let new_dir = args.peekable().peek().map_or("/", |x| *x);
                let root = Path::new(new_dir);
                if let Err(e) = env::set_current_dir(&root) {
                    eprintln!("{}", e);
                }
            },
            command => {
                let mut child = Command::new(command)
                    .args(args)
                    .spawn()
                    .unwrap();

                child.wait();
            }
        }
    }
}
```

## Error Handling

If you've been following along, you've probably already noticed that the shells above will crash if you input a command which does not exist. In the version below, that is handled gracefully, by printing an error to the user and then allowing them to enter another command. 

Since entering a bad command was acting as an easy way to quit the shell, I've also implemented another shell built-in, the `exit` command. 

```rust
fn main(){
    loop {
        print!("> ");
        stdout().flush();

        let mut input = String::new();
        stdin().read_line(&mut input).unwrap();

        let mut parts = input.trim().split_whitespace();
        let command = parts.next().unwrap();
        let args = parts;

        match command {
            "cd" => {
                let new_dir = args.peekable().peek().map_or("/", |x| *x);
                let root = Path::new(new_dir);
                if let Err(e) = env::set_current_dir(&root) {
                    eprintln!("{}", e);
                }
            },
            "exit" => return,
            command => {
                let child = Command::new(command)
                    .args(args)
                    .spawn();

                // gracefully handle malformed user input
                match child {
                    Ok(mut child) => { child.wait(); },
                    Err(e) => eprintln!("{}", e),
                };
            }
        }
    }
}
```

## Pipes

It would be difficult to be productive in a shell which didn't include pipes. If you aren't familiar with this feature, the `|` character is used to tell the shell to redirect the output of the first command into the input of the second command. For example, running the command `ls | grep Cargo` triggers the following set of actions:

1. `ls` will list all files in the current directory
2. The shell will pipe the above list of files to `grep`
3. `grep` will filter the list and output only files which contain the string `Cargo`

This final iteration of our shell includes very basic support for pipes. For an introduction to many other things pipes and IO redirection can do, check out [this article](https://robots.thoughtbot.com/input-output-redirection-in-the-shell).

```rust
fn main(){
    loop {
        print!("> ");
        stdout().flush();

        let mut input = String::new();
        stdin().read_line(&mut input).unwrap();

        // must be peekable so we know when we are on the last command
        let mut commands = input.trim().split(" | ").peekable();
        let mut previous_command = None;

        while let Some(command) = commands.next()  {

            let mut parts = command.trim().split_whitespace();
            let command = parts.next().unwrap();
            let args = parts;

            match command {
                "cd" => {
                    let new_dir = args.peekable().peek()
                        .map_or("/", |x| *x);
                    let root = Path::new(new_dir);
                    if let Err(e) = env::set_current_dir(&root) {
                        eprintln!("{}", e);
                    }

                    previous_command = None;
                },
                "exit" => return,
                command => {
                    let stdin = previous_command
                        .map_or(
                            Stdio::inherit(),
                            |output: Child| Stdio::from(output.stdout.unwrap())
                        );

                    let stdout = if commands.peek().is_some() {
                        // there is another command piped behind this one
                        // prepare to send output to the next command
                        Stdio::piped()
                    } else {
                        // there are no more commands piped behind this one
                        // send output to shell stdout
                        Stdio::inherit()
                    };

                    let output = Command::new(command)
                        .args(args)
                        .stdin(stdin)
                        .stdout(stdout)
                        .spawn();

                    match output {
                        Ok(output) => { previous_command = Some(output); },
                        Err(e) => {
                            previous_command = None;
                            eprintln!("{}", e);
                        },
                    };
                }
            }
        }

        if let Some(mut final_command) = previous_command {
            // block until the final command has finished
            final_command.wait();
        }

    }
}
```

## Conclusion

In less than 100 lines of Rust we've created a shell which would be usable for many day to day tasks, but a real shell has many more features. The GNU website has an online manual for the bash shell, including this list of [shell features](https://www.gnu.org/software/bash/manual/html_node/Basic-Shell-Features.html#Basic-Shell-Features) which is a great place to start looking into the more advanced functionality. 

Please note that this was a learning project for me, and in cases where there was a trade-off between simplicity and robustness I most often chose simplicity. 

This shell is [available on GitHub](https://github.com/JoshMcguigan/bubble-shell). The latest commit as of this writing is [a47640](https://github.com/JoshMcguigan/bubble-shell/tree/a6b81d837e4f5e68cf0b72a4d55e95fb08a47640). Another Rust shell learning project that you may be interested in is [Rush](https://github.com/psinghal20/rush).
