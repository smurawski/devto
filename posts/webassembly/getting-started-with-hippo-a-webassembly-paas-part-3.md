---
title: Getting Started with Hippo - a WebAssembly PaaS (Part 3)
description: Converting an Existing Application   With the understanding we’ve built of the runtime...
tags: 'rust, webassembly, beginners'
published: true
id: 901233
date: '2021-11-17T22:43:16.288Z'
---

# Converting an Existing Application

With the understanding we’ve built of the runtime environment, I feel ready to start porting a simple CLI I’ve built in Rust to run in WebAssembly as a service hosted in Hippo. [The project we’ll start with is J2Y(https://github.com/smurawski/j2y/tree/1-getting-started) – which is a little Rust application that converts JSON to YAML or YAML to JSON. We’ll adapt this to, depending on the target, either be a CLI or a WebAssembly binary to run in WAGI. The heavy lifting of the conversion is done by the [serde-json](https://github.com/serde-rs/json) and the [serde-yaml](https://github.com/dtolnay/serde-yaml) crates.

We’ll start out by taking a look at the existing application. We are still using VS Code with the Remote-SSH extension to connect to our dev server. The main function in main.rs ( https://github.com/smurawski/j2y/blob/1-getting-started/src/main.rs ) module is the primary entry point for the application. The function sets up the CLI experience and resolves the incoming parameters and arguments. The application then reads the source content. Based on the direction of the conversion, it passes off the text it read to the desired conversion function. Finally, the application writes the output file in the desired serialization.

![Visual studio code showing the main function in main.rs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iw68ac3ebhd25khy7l0r.png)

## Goals

This isn’t a treatise on how to properly structure a Rust application, but mostly to show how we can adapt an existing application to a WebAssembly module to run in Hippo. There will be plenty of opportunities to clean up the code base and refactor to make it more maintainable over time.

For our web service, we’ll take a posted body of either JSON or YAML and return the opposite. We’ll use the Content-Type header to help us determine the direction of the conversion. Then, then we’ll return a response with the appropriate file content as the body.

## Evolving the App

### Compile to WASM

We’ll start by adjusting our main function to behave conditionally based on whether or not it’s built as a WASM module. The `cfg!` macro helps us here, as we can create conditions in our code based on the target family (or operating system, or target architecture) we compile for. On anything that isn’t WASM, we want to build our CLI as normal. I’ve stubbed out the return of the parameters for the function and added [a conditional based on our `target_family`](https://github.com/smurawski/j2y/blob/2-target-family/src/main.rs#L19).

```rust
    let (verbose, input_file, output_file, source_format) = if cfg!(target_family = "wasm") {
        get_wasm_args()
    } else {
        get_cli_args()
    };
```

![Editor with a conditional based on the target_family compiled for](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5ae0928trn6oyh1nam98.png)

Again, ideally we’d refactor to a common branch point or push the decision logic further down into the functions like `read_content`, but for this example, we’ll keep all the branching up in main to keep it visible.

### Working with Input from WAGI

Let’s create a module to contain our WASM and WAGI specific behaviors. We’ll start with a function to create the tuple of arguments similar to what the `get_cli_args` function does. Then we’ll adjust the stubbed out response in main.rs to call our new function. You can see [the full changeset here](https://github.com/smurawski/j2y/commit/82730cf2272ca156e1fbdd5e835ad934293ec6ab) and the [current state of the full project](https://github.com/smurawski/j2y/tree/3-wasm-args).

```rust
// in wasm.rs
pub fn get_wasm_args() -> (bool, String, String, SourceFormat) {
    // we'll default to verbose being false as it writes stdout,
    // and stdout is what becomes our response back
    let verbose = false;
    // the input and output files are not used.
    // this is a good point for some future refactoring
    let input_file = String::new();
    let output_file = String::new();
    let input_content_type = env::var("HTTP_CONTENT_TYPE")
        .expect("The Content-Type header was not specified. Unable to convert the source content.");
    let source_format = get_source_format(&input_content_type);
    (verbose, input_file, output_file, source_format)
}
```

![editor with wasm.rs and the get_wasm_args function](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mgo615nvjjclngwondc0.png)

As we learned in our experimentation in the last post, we can inspect our environment to get the incoming headers. Next, [we can add a function to read from standard in, which is where the body of the incoming request will be available and update `main.rs`](https://github.com/smurawski/j2y/tree/4-read-from-wagi).

```rust
// in wasm.rs
pub fn read_wagi_content() -> String {
    let mut input_content = String::new();
    stdin()
        .read_to_string(&mut input_content)
        .expect("Failed to read from standard in.");
    input_content
}
```

```rust
// in main.rs
    let contents = if cfg!(target_family = "wasm") {
        read_wagi_content()
    } else {
        read_content(&input_file, verbose)
    };
```

![editor with wasm.rs and the read_wagi_content function](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3ym5c3njntyl0m8idahg.png)

The last tweak we have to make is our output. Rather than writing to a file, we’ll write to standard out. Since we are following the CGI convention, we’ll write our our headers, then a blank line, then our body. [We’ll update main.rs to call `write_wagi_output`](https://github.com/smurawski/j2y/tree/5-write-to-wagi).

```rust
// in wasm.rs
pub fn write_wagi_output(output: &str, source_format: &SourceFormat) {
    println!("Content-Type: {}", get_output_format(source_format));
    println!();
    println!("{}", output);
}
```

```rust
// in main.rs
    if cfg!(target_family = "wasm") {
        write_wagi_output(&output_content, &source_format);
    }
    else {
        write_content(&output_file, output_content, verbose).expect("Failed to write the output file");
    }
```

### Connecting with Hippo and Bindle

Now, how do we get this app into Hippo? We can use `yo wasm` again! We can use that to create our project in Hippo, create the `HIPPOFACTS` file, and prep us to deploy to Bindle. There are a few files that we’ll get a conflict on, and we can just ask `yo wasm` to leave our existing files alone.

> (HINT: if you need your environment variables again before running `yo wasm`, check `~/output.txt`.)

After `yo wasm` is run, we'll see our app configured in Hippo.

![new app j2y available in the hippo dashboard](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/geuo14afdovm5be1yccu.png)

As before, we’ll build our app for the wasm32-wasi target. We can use the Hippo CLI to push the artifact to Bindle and then into Hippo.

```bash
cargo build --release --target wasm32-wasi
hippo push -k .
```

Now that, we’ve deployed the application, we can see in the Hippo dashboard that it’s running at https://admin.j2y.localhost:5003. Let’s make sure that 5003 is being forwarded by our VS Code Remote session.

![Visual studio code showing port forwarding configuration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j2wxhsdo0hibff8zd3os.png)

### Testing our App

Now that I have that port forwarded, I can start to test that my service behaves as expected. ([If you are playing along, we are here](https://github.com/smurawski/j2y/tree/6-HIPPOFACTS-and-test-scripts)) Let’s start out with some curl commands. I've created a few sample data files in a `tests` directory ([a json file](https://github.com/smurawski/j2y/blob/6-HIPPOFACTS-and-test-scripts/tests/test.json) and [a yaml file](https://github.com/smurawski/j2y/blob/6-HIPPOFACTS-and-test-scripts/tests/test.yml)) that we'll use to feed in as the `POST` body.

```bash
cd tests/
url="https://admin.j2y.localhost:5003/"
# I expect this to fail because there is no application header
curl -vv --request POST -k --data @test.json $url
# This should succeed and return some YAML
curl --header "Content-Type: application/json" --request POST -k --data @test.json $url
```

![terminal in editor running successful curl command and getting back YAML](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6i9dhhxf9tebdq4tozxo.png)

```
# This should succeed and return some JSON
curl --header "Content-Type: application/yaml" --request POST -k --data @test.yml $url

# Uh oh... Let’s look closer
curl -vv --header "Content-Type: application/yaml" --request POST -k --data @test.yml $url
```

![terminal in editor running curl command that fails with a 500 error](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2fsxbuv0n0jm1ccaljnc.png)

### Dealing with Errors

We can see that something failed in the conversion from YAML to JSON. Let’s try our CLI and make sure we didn’t break anything.

```bash
cargo run -- -s YAML test.yml output.json
cat output.json
```

![terminal in editor running cargo run to execute locally](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7opqin0n2e3jmts9xoqe.png)

Well, that seemed to work ok. That doesn't help. I was hoping that something was wrong with my YAML file.

So, how can we figure out what’s going wrong? We could introduce some custom output to help us understand where the failure was. Currently, if anything fails in the conversion between serialization formats the application panics causing a failure and a 500 response, but no helpful information. Let’s change the behavior and rather than unwrapping the conversion results in our main function, let’s pass that into our functions that actually return the results. Because each of the deserializations returns a different error (`serde_yaml::Error` or `serde_json::Error`, let’s add a custom error of our own to wrap those in `converter.rs`.

```rust
// in converter.rs
pub struct Error {
    pub message: String,
    pub detail: String,
    pub status_code: usize,
    pub source_content: String,
}

impl fmt::Display for Error {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

impl fmt::Debug for Error {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(
            f,
            "{}\n{}\nReturn Status Code: {}\nSource Content: \n{}",
            self.message, self.detail, self.status_code, self.source_content
        )
    }
}
```

![editor showing new struct for an Error in converter.rs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nv8i8nbccd6a96pbcva9.png)

This custom error will allow us to capture different points of the failure and give us information to troubleshoot the problem. We can then wire it up into the conversion functions. You’ll see that we won’t need to change our main function just yet. ([Find the full code for this checkpoint here](https://github.com/smurawski/j2y/tree/7-add-custom-error).)

```rust
// in converter.rs
pub fn convert_json_to_yaml(json_str: &str, verbose: bool) -> Result<String, Error> {
    // Parse the string of json data into serde_yaml::Value.
    let v: serde_yaml::Value = match serde_json::from_str(json_str) {
        Ok(v) => v,
        Err(e) => {
            let wrapped_error = Error {
                message: "Failed to read the content as JSON.".to_string(),
                detail: format!("{:?}", e),
                status_code: 406,
                source_content: json_str.to_string(),
            };
            return Err(wrapped_error);
        }
    };
    let yaml_string = match serde_yaml::to_string(&v) {
        Ok(v) => v,
        Err(e) => {
            let wrapped_error = Error {
                message: "Failed to convert the JSON content into YAML.".to_string(),
                detail: format!("{:?}", e),
                status_code: 500,
                source_content: json_str.to_string(),
            };
            return Err(wrapped_error);
        }
    };

    if verbose {
        println!("\nAfter YAML conversion: \n");
        println!("{}", &yaml_string);
    }

    Ok(yaml_string)
}
```

```rust
// in converter.rs
pub fn convert_yaml_to_json(yaml_str: &str, verbose: bool) -> Result<String, Error> {
    // Parse the string of json data into serde_yaml::Value.
    let v: serde_json::Value = match serde_yaml::from_str(yaml_str) {
        Ok(v) => v,
        Err(e) => {
            let wrapped_error = Error {
                message: "Failed to read the content as YAML.".to_string(),
                detail: format!("{:?}", e),
                status_code: 406,
                source_content: yaml_str.to_string()
            };
            return Err(wrapped_error);
        }
    };
    let json_string = match serde_json::to_string(&v) {
        Ok(v) => v,
        Err(e) => {
            let wrapped_error = Error {
                message: "Failed to convert the YAML content into JSON.".to_string(),
                detail: format!("{:?}", e),
                status_code: 500,
                source_content: yaml_str.to_string()
            };
            return Err(wrapped_error);
        }
    };

    if verbose {
        println!("\nAfter YAML conversion: \n");
        println!("{}", &json_string);
    }

    Ok(json_string)
}
```

After we add that custom error, we can wire up our output methods to use it more effectively and provide us some useful output from our Hippo service. You can see [the full changes here](https://github.com/smurawski/j2y/tree/8-custom-error-message).

```rust
// in wasm.rs
pub fn write_wagi_output(output_result: Result<String, Error>, source_format: &SourceFormat) {
    let mut content_type = get_output_format(source_format);
    let mut status = 200;
    let output = match output_result {
        Ok(output) => output,
        Err(e) => {
            content_type = "text/plain".to_string();
            status = e.status_code;
            format!("{:?}", e)
        }
    };

    println!("Content-Type: {}", content_type );
    println!("Status: {}", status);
    println!();
    println!("{}", output);
}
```

```rust
// in cli.rs
pub fn write_content(file_path: &str, output_result: Result<String, Error>, verbose: bool) -> io::Result<()> {
    if verbose {
        println!("\nWriting: {} \n", file_path);
    }
    let output_content = output_result.unwrap();
    let mut file = File::create(file_path).expect("Failed to create the output file.");
    file.write_all(output_content.into_bytes().as_ref())
}
```

```rust
// in main.rs

      let output_content =
        match source_format {
            SourceFormat::YAML => convert_yaml_to_json(&contents, verbose),
            SourceFormat::JSON => convert_json_to_yaml(&contents, verbose),
        };
```

Now, we can build and publish our app, and then re-rerun our test.

```bash
cargo build --release --target wasm32-wasi
hippo push -k .

cd tests/
curl --header "Content-Type: application/yaml" --request POST -k --data @test.yml $url
```

![terminal in editor with failed curl command with our custom error data.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fb77wwkiudlo54twofxj.png)

Now, we can see why it failed! When we passed in the YAML file, it appears that our line breaks have been lost! Fortunately, we can fix that easily by using a different switch ( `--data-binary` rather than `--data` ). With that change, we can see our conversion from YAML to JSON works as well.

![terminal in editor with a successful curl command and JSON response](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xql0wmzmfpnvgfb2suaf.png)

## Wrapping up

We now have our application running as a WASM module as a service hosted on our Hippo server. From here we can continue to improve our application, explore the different hosting options inside of Hippo – testing the different deployment based on the tags assigned for example, or dive deeper into the WAGI runtime.

If you’d like to try to migrate the app along the same path I did, you can start from https://github.com/smurawski/j2y/tree/1-getting-started

Have fun with WASM and Hippo!
