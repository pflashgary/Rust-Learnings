# [Serilized Gantt Chart](https://github.com/pflashgary/gantt-chart)

A Gantt chart is a type of bar chart that illustrates a project schedule, named after Henry Gantt who invented this chart in the early 1900s. It is used to represent the timeline of a project or multiple projects, and displays the tasks or activities involved, their duration, and the dependencies between them. In software development, it is often used to track the progress of building features for a product, with each task or activity representing a step in the development process. The horizontal axis of a Gantt chart represents time, and the vertical axis represents tasks or activities. The chart helps project managers and teams to visualize the work that needs to be done, estimate the time it will take, and track progress towards completion.

Our program aims to convert the components of a Gannt diagram into a JSON representation. This allows for the data to be more easily accessible and processed for further analysis and use.


### Introduction

I will discuss a problem of representing a set of features and their children in a structured manner. The goal is to convert this information into a JSON representation.

Problem Statement:

Given a set of features defined by the following lines:

_[start_date] [end_date] [program_id] [progress_status] [assigned_team] [parent_feature]->[feature]_

```txt
2023-01-01 2023-12-31 program1 InProgress TeamA null->ProductivitySuite
2023-01-01 2023-06-30 program1 Complete TeamB ProductivitySuite->Email
2023-01-01 2023-04-30 program1 Complete TeamB Email->EmailSearch
2023-05-01 2023-06-30 program1 Complete TeamB Email->EmailFilters
```
**Warning**: The code uses `chrono::DateTime`. I need to change to `chrono:Date` to ignore the time component.

The task is to convert this information into a JSON representation that includes all the features and their children.

The input data will be stored in a text file, and the solution should be able to read this file and convert it into a JSON representation. The resulting JSON should be organized in a hierarchical manner, with the parent feature/module being the root, and its children being the sub-features.

```json
{
    "id": "program1",
    "root": {
        "feature": "ProductivitySuite",
        "start": "2023-01-01T00:00:00.000Z",
        "end": "2023-12-31T00:00:00.000Z",
        "subfeatures": [
            {
                "feature": "Email",
                "start": "2023-01-01T00:00:00.000Z",
                "end": "2023-06-30T00:00:00.000Z",
                "subfeatures": [
                    {
                        "feature": "EmailSearch",
                        "start": "2023-01-01T00:00:00.000Z",
                        "end": "2023-04-30T00:00:00.000Z",
                        "subfeatures": [],
                        "progress_status": "Complete",
                        "assigned_team": "TeamB"
                    },
                    {
                        "feature": "EmailFilters",
                        "start": "2023-05-01T00:00:00.000Z",
                        "end": "2023-06-301T00:00:00.000Z",
                        "subfeatures": [],
                        "progress_status": "Complete",
                        "assigned_team": "TeamB"
                    }
                ],
                "progress_status": "Complete",
                "assigned_team": "TeamB"
           }
        ],
        "progress_status": "Complete",
        "assigned_team": "TeamA"
    }
}
```

## Workspace members

### Program Ingester

#### `input.rs`

The input crate is responsible for defining the input structs and implementing the required traits for reading and parsing the input data. The crate contains two main structs, `Ingester` and `RawFeature`.


##### Ingester

The `Ingester` struct has a single public field features, which is a vector of `RawFeatures`. `Ingester` implements the `TryFrom` trait, which allows converting an instance of `BufReader<R>` (where `R` implements the Read trait) into an instance of `Ingester`.

The implementation of the `TryFrom` trait for `Ingester` reads lines from the `BufReader<R>` using the `read_line` method, removes the trailing newline character, converts the line to a `RawFeature` using the `from_str` method, and pushes the `RawFeature` to the features vector. If any errors occur during the reading or conversion process, the implementation returns an `Err` variant of the Result, with the error being of type `crate::errors::ProgramIngesterError`.

**Note** The reason we use `while reader.read_line()` instead of iterating over the lines with `map` is because the latter allocates a string for each iteration, while with a dumb loop we can instead use string buffer that is allocated once, and cleared after each loop.


```rust
pub struct Ingester {
    pub features: Vec<RawFeature>,
}

impl<R: Read> TryFrom<BufReader<R>> for Ingester {
    type Error = crate::errors::ProgramIngesterError;

    fn try_from(mut reader: BufReader<R>) -> Result<Self, Self::Error> {
        let mut features = vec![];
        let mut buf = String::new();

        // It's more efficient to allocate a single string buffer and loop
        // over reader.read_line(), rather than using reader.lines().map()
        // which will allocate a new String on each iteration
        while reader.read_line(&mut buf)? > 0 {
            {
                // remove the trailing \n
                let line = buf.trim_end();
                let feature = RawFeature::from_str(line)?;
                features.push(feature);
            }
            buf.clear();
        }
        Ok(Ingester { features })
    }
}

```

##### RawFeature

The `RawFeature` struct contains 7 fields, including:
- ID of the node
- parent_id (if set to `None`, then it is a root feature)
- program_id
- progress_status (Complete or In Progress)
- assigned_team generating the feature
- start_date
- end_date

##### Implementations
There are implementations for the `TryFrom` and `FromStr` traits for the `RawFeature` struct.
- The `TryFrom` implementation allows creating a `RawFeature` instance from a `String`
- The `FromStr` implementation allows creating a `RawFeature` instance from a string slice (`&str`) and tries to turn a program log line into a feature by parsing the string slice into its component parts. If the string slice doesn't have the expected format or can't be parsed into a `RawFeature`, then it returns an error.

The `FromStr` implementation has a `from_str` method which takes a string `s` as input and returns a `Result` containing either an instance of the `RawFeature` type or an error.

The input string `s` is first trimmed and split into a vector of string slices `parts` using the `split` method. If the resulting vector `parts` doesn't have 6 elements, the code returns an error indicating an invalid input string.

Next, the code takes the last element of the `parts` vector and splits it into two parts using the `->` separator, resulting in a `feature_ids` vector. If the `feature_ids` vector doesn't have 2 elements, the code returns an error indicating an invalid input string.

If all the checks have passed, the code creates an instance of the `RawFeature` type and returns it as the result of the function.

```rust

#[derive(Debug, PartialEq, Eq)]
pub struct RawFeature {
    /// This node's ID
    pub id: String,

    /// The parent feature
    ///
    /// If it is set to `None`, then this is a root feature
    pub parent_id: Option<String>,

    /// Program ID
    pub program_id: String,

    /// progress status (Complete, In Progress)
    pub progress_status: String,

    /// The name of the assigned team generating the feature
    pub assigned_team: String,

    /// The Feature Start Time
    pub start_date: chrono::DateTime<FixedOffset>,

    /// Feature End Time
    ///
    /// **Note**: This could be later than child features if the child features are asynchronous.
    pub end_date: chrono::DateTime<FixedOffset>,
}

impl RawFeature {
    pub fn is_root(&self) -> bool {
        self.parent_id.is_none()
    }
}

impl TryFrom<String> for RawFeature {
    type Error = crate::errors::ProgramIngesterError;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        RawFeature::from_str(value.as_str())
    }
}

impl FromStr for RawFeature {
    type Err = crate::errors::ProgramIngesterError;

    /// Try to turn a program log line into a feature
    ///
    /// Example: `2016-10-20T12:43:34.000Z 2016-10-20T12:43:35.000Z program1 back-end-3 ac->ad`
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let parts: Vec<&str> = s.trim().split(" ").collect();
        // We should have 6 parts.
        if parts.len() != 6 {
            return Err(ProgramIngesterError::InvalidProgramInput(format!(
                "The feature '{s}' needs to have 6 parts, start, end, program, progress_status, assigned_team, feature-relation"
            )));
        }

        // If so, then we need to split the last one part
        let last_part = parts
            .last()
            .expect("we checked that there are 6 parts earlier");
        let feature_ids: Vec<&str> = last_part.split("->").collect();

        if feature_ids.len() != 2 {
            return Err(ProgramIngesterError::InvalidProgramInput(format!("The feature-relation '{s}' needs to have 2 parts")));
        }

        Ok(RawFeature {
            id: feature_ids.last().unwrap().to_owned().into(),
            parent_id: match feature_ids.first().unwrap().to_owned() {
                "null" => None,
                id => Some(id.into()),
            },
            start_date: DateTime::parse_from_rfc3339(parts.first().unwrap().to_owned())?,
            end_date: DateTime::parse_from_rfc3339(parts.get(1).unwrap().to_owned())?,
            program_id: parts.get(2).unwrap().to_owned().into(),
            progress_status: parts.get(3).unwrap().to_owned().into(),
            assigned_team: parts.get(4).unwrap().to_owned().into(),
        })
    }
}

```

##### `FeatureDataAndChildren`

The `FeatureDataAndChildren` struct holds two pieces of data:

- `feature_data`: an `Option` of a reference to a `RawFeature` struct. It's an `Option` because it is only updated to `Some` when that data is found (which might be later than when the `RawFeature` is created). This means that `feature_data` could either contain a reference to a `RawFeature` instance, or it could be `None`, indicating that there is no `RawFeature` instance associated with this struct.

- `children`: a Vec (vector) of `FeatureID` values. This is a list of child feature IDs that can be used to look up related information in the `FeatureMap`.

`FeatureMap` is a type alias for a HashMap (hash map) data structure. A HashMap is a collection of key-value pairs, where the keys are of type `FeatureID` (which is defined as a type alias for String) and the values are of type `FeatureDataAndChildren`. This type alias makes it more expressive and easier to use, as the type `FeatureMap` is more meaningful and understandable than the underlying type HashMap.

'a is a lifetime annotation. In Rust, lifetimes are a way of expressing the relationship between references. They ensure that references are used in a safe and correct way by preventing references from pointing to data that has been dropped.

```rust
#[derive(Debug)]
pub struct FeatureDataAndChildren<'a> {
    // This refers to existing data already ingested
    pub feature_data: Option<&'a RawFeature>,

    // These are the child feature IDs which can then be looked up in the FeatureMap (HashMap)
    pub children: Vec<FeatureID>,
}
// alias types to make usage simpler and more expressive
pub type FeatureID = String;
pub type FeatureMap<'a> = HashMap<FeatureID, FeatureDataAndChildren<'a>>;
```

#### `errors.rs`

##### ProgramIngesterError

This code defines an error enum for the ProgramIngester module, named `ProgramIngesterError`. The enum has three variants:

###### InvalidProgramInput
This variant is used when the input to the program is not valid, and it carries a string message describing the error.

###### InvalidTimestamp
This variant is used when the timestamp in the input cannot be parsed, and it carries an underlying error of type `chrono::ParseError`.

###### IoError
This variant is used when an I/O operation fails, and it carries an underlying error of type `io::Error`.

The `Error` trait and the `#[derive(Error, Debug)]` attribute are from the `thiserror` crate. `thiserror` allows to handle custom errors as well as wrapping errors produced by other crates (to make using `?` possible).


```rust
use std::io;

use thiserror::Error;

#[derive(Error, Debug)]
pub enum ProgramIngesterError {
    #[error("The program input is not valid: {0}")]
    InvalidProgramInput(String),

    #[error("The timestamp could not be parsed")]
    InvalidTimestamp {
        #[from]
        source: chrono::ParseError,
    },

    #[error("The IO operation failed: {source}")]
    IoError {
        #[from]
        source: io::Error,
    },
}
```

#### `output.rs`
##### Feature and Serialization

The `Feature` struct represents a feature in a software development project. It has the following fields:
- `id`: a string identifier for the feature
- `progress_status`: a string indicating the progress status of the feature
- `assigned_team`: a string indicating the team responsible for the feature
- `start_date`: a `chrono::DateTime<FixedOffset>` indicating the start date of the feature
- `end_date`: a `chrono::DateTime<FixedOffset>` indicating the end date of the feature
- `subfeatures`: a vector of `Feature` objects, representing subfeatures of the current feature

The struct is decorated with `#[derive(Debug, Serialize, Clone)]`, which uses Rust's "derive" macro to automatically generate implementations for the `Debug`, `Serialize`, and `Clone` traits. This means that instances of `Feature` can be debugged, serialized (converted to a format like JSON or BSON), and cloned (duplicated).

The line `#[serde(rename = "feature")]` uses Serde's "serde" attribute to specify that the `id` field should be renamed to `"feature"` when serializing or deserializing instances of `Feature`. The line `#[serde(serialize_with = "odered_features")]` uses Serde's "serde" attribute to specify that the `subfeatures` field should be serialized using a custom serialization function named `odered_features`. This allows for custom logic to be used when serializing the `subfeatures` field.


```rust
#[derive(Debug, Serialize, Clone)]
pub struct Feature {
    #[serde(rename = "feature")]
    pub id: String,
    pub progress_status: String,
    pub assigned_team: String,
    pub start_date: chrono::DateTime<FixedOffset>,
    pub end_date: chrono::DateTime<FixedOffset>,
    #[serde(serialize_with = "odered_features")]
    pub subfeatures: Vec<Feature>,
}
```

###### Custom Serializer

The custom serializer `odered_features` is defined as follows:

```rust
fn odered_features<S>(value: &[Feature], serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    let mut value = value.to_owned();
    value.sort_by_key(|feature| feature.start_date);

    value.serialize(serializer)
}

```

The function takes in a slice of `Feature` objects, `value`, and a Serde serializer, `serializer`. The function sorts the input slice of `Feature` objects by the `start_date` field and then serializes the sorted slice using the provided `serializer`. The function returns a `Result` type that represents the outcome of the serialization process. If the serialization is successful, the result will contain the serialized value of type `S::Ok`. If an error occurs during serialization, the result will contain an error value of type `S::Error`. The function uses the `where` clause to specify that the type of the `serializer` must implement the `Serializer` trait. This allows the function to be used with any serializer that implements this trait, making it more flexible and reusable. The method `to_owned()` is a method that creates a new owned value (a deep copy) from a borrowed value, such as a reference. In this case, the method is used to convert the input slice of `Feature` objects, `value`, into an owned vector of `Feature` objects. This is necessary because the sorting operation needs to modify the contents of the vector, and a reference to the original slice cannot be modified. By creating an owned copy, the original data remains unchanged, and the sort operation can be performed on the copy.


##### Program
This struct has two fields: `id` of type String and `root` of type `Feature`.

```rust

#[derive(Debug, Serialize, PartialEq, Clone)]
pub struct Program {
    pub id: String,
    pub root: Feature,
}
```

##### ProgramGraph

This struct has one field: programs of type `Vec<Program>`.

```rust

#[derive(Debug, Default, Serialize, Clone)]
pub struct ProgramGraph {
    #[serde(serialize_with = "odered_programs")]
    pub programs: Vec<Program>,
}

```

###### Serialization with `odered_programs`

The `#[serde(serialize_with = "odered_programs")]` line is a Serde attribute that specifies the custom serialization function for the programs field. The odered_programs function sorts the programs by their root.start_date value before serializing them.

```rust
fn odered_programs<S>(value: &[Program], serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    let mut value = value.to_owned();
    value.sort_by_key(|program| program.root.start_date);

    value.serialize(serializer)
}

```

###### Partial Equality with `PartialEq` Implementation

The `PartialEq` implementation for `ProgramGraph` sorts the programs field by their root.start_date value before checking for equality.

```rust
impl PartialEq for ProgramGraph {
    fn eq(&self, other: &Self) -> bool {
        // sort the programs by root start date before equality check
        let mut these_programs = self.programs.clone();
        these_programs.sort_by_key(|program| program.root.start_date);
        let mut those_programs = other.programs.clone();
        those_programs.sort_by_key(|program| program.root.start_date);

        these_programs == those_programs
    }
}
```

###### Transformation from `RawFeature` to `ProgramGraph`

The code is transforming a vector of `RawFeature` objects into a `ProgramGraph` object. The transformation starts by building a mapping of parent to children feature IDs, which is done by upserting each feature and its parent. If the feature or its parent already exists in the mapping, it is updated, otherwise it is inserted with the given information.

Once the mapping is built, the code filters out the root features, which are the features that don't have a parent. For each root feature, the code creates a Program object by resolving its subfeatures and then pushes it into the ProgramGraph object.

The code uses a helper function, `resolve_subfeatures`, to resolve the subfeatures of a feature. It takes a list of child feature IDs and the mapping and returns a list of Feature objects by filtering the mapping and transforming the filtered values into `Feature` objects.

The impl `From` block is an implementation of the `From` trait, which allows a type conversion from `Vec<RawFeature>` to `ProgramGraph`. The implementation makes use of the helper function `resolve_subfeatures` to resolve the subfeatures of each feature in the vector of `RawFeature` objects and construct the `ProgramGraph` object.

```rust
/// Transform a vector of [`RawFeature`] into a [`ProgramGraph`]
impl From<Vec<RawFeature>> for ProgramGraph {
    fn from(value: Vec<RawFeature>) -> Self {
        // given a list of child feature IDs, recursively resolve the child features
        #[tracing::instrument(skip(mappings))]
        fn resolve_subfeatures(child_ids: Vec<FeatureID>, mappings: &FeatureMap) -> Vec<Feature> {
            //tracing::debug!("resolving children: {child_ids:?}");
            mappings
                //.values()
                .iter()
                //.filter(|&value| {
                .filter(|(feature_id, value)| {
                    if let Some(feature_data) = value.feature_data {
                        child_ids.contains(&feature_data.id) // todo: use feature_id, remove if/else
                    } else {
                        tracing::debug!(feature_id, "no feature_data found");
                        false
                    }
                })
                //.filter_map(|value| {
                .filter_map(|(feature_id, value)| {
                    tracing::debug!(feature_id, "...");
                    value.feature_data.map(|feature_data| Feature {
                        id: feature_data.id.clone(),
                        start_date: feature_data.start_date,
                        end_date: feature_data.end_date,
                        assigned_team: feature_data.assigned_team.clone(),
                        progress_status: feature_data.progress_status.clone(),
                        subfeatures: resolve_subfeatures(value.children.clone(), mappings),
                    })
                })
                .collect()
        }

        // a place to hold the parent -> children references
        let mut mappings = FeatureMap::new();

        for feature in value.iter() {
            // 1. Upsert this feature in the mappings:
            //    - If it exists already (a child created it to add itself to children, see #2): Set the feature_data to Some(...)
            //    - If it doesn't exist: Insert a new entry with the feature_data set to Some(...) and the children as an empty Vec
            // 2. Upsert the parent feature in the mappings
            //    - If it exists already (created by #1, or by another child feature in #2): Only push this id as a child.
            //    - If it doesn't exist: Insert a new entry with the feature_data set to None and the children as a Vec with one entry (this feature id)
            // By the end, there should be no feature_data values set to None

            match mappings.get_mut(&feature.id) {
                Some(mapping) => mapping.feature_data = Some(feature),
                None => {
                    mappings.insert(
                        feature.id.clone(),
                        FeatureDataAndChildren {
                            feature_data: Some(feature),
                            children: vec![],
                        },
                    );
                }
            };

            if let Some(parent_id) = feature.parent_id.clone() {
                match mappings.get_mut(&parent_id) {
                    Some(mapping) => mapping.children.push(feature.id.to_owned()),
                    None => {
                        mappings.insert(
                            parent_id,
                            FeatureDataAndChildren {
                                feature_data: None,
                                children: vec![feature.id.to_owned()],
                            },
                        );
                    }
                }
            };
        }

        let roots = mappings.values().filter(|&value| {
            // * is_some_and is way handier in the filter, but it's a feature only available in rust-nightly
            // value
            //     .feature_data
            //     .is_some_and(|&feature_data| feature_data.is_root())
            // But alas, we have to do it the uglier way for now:
            if let Some(feature_data) = value.feature_data {
                feature_data.is_root()
            } else {
                false
            }
        });

        let mut graph = ProgramGraph::default();
        // this will loop over roots to build Program structs, and push them into the Graph
        for root in roots {
            if let Some(feature_data) = root.feature_data {
                let program = Program {
                    id: feature_data.program_id.clone(),
                    root: Feature {
                        id: feature_data.id.clone(),
                        start_date: feature_data.start_date,
                        end_date: feature_data.end_date,
                        assigned_team: feature_data.assigned_team.clone(),
                        progress_status: feature_data.progress_status.clone(),
                        subfeatures: resolve_subfeatures(root.children.clone(), &mappings),
                    },
                };
                graph.programs.push(program);
            } else {
                todo!("warn about missing feature_data");
            }
        }

        graph
    }
}

```
#### `lib`
This `lib` crate is the root of the library, pulling in the previously mentioned modules and leaves implementation to another crate (eg: the `cli`, or anyone else that want's to implement it different, like in a web server instead of CLI).

### CLI

The command line interface (CLI) tool ingests data, builds a graph and outputs it to the terminal. The following are the high-level steps:

1. Initialize tracing: A tracing-subscriber is set up to log messages to standard error (stderr). The log level is either set by the RUST_LOG environment variable, or falls back to info if the environment variable is not set.

2. Read CLI arguments: The command line arguments passed to the program are collected into a vector of strings. If there are more than 2 arguments, a warning is logged that the additional arguments will be ignored.

3. Build the ingester: The ingester reads input from either a file specified in the command line arguments, or from standard input (stdin) if no file is specified. The ingester is built using the Ingester struct, which provides a method to parse features from the input source.

4. Build the graph: A ProgramGraph is built from the ingester's features. The ProgramGraph is a collection of Programs, and each Program has an id and a root Feature.

5. Output the graph: The ProgramGraph is output to the terminal using the println! macro and the Debug format.


```rust
use std::{
    env,
    fs::File,
    io::{self, BufReader},
};

use program_ingester::{input::Ingester, output::ProgramGraph};
use tracing_subscriber::layer::SubscriberExt;

fn main() -> anyhow::Result<()> {
    // Create a stdout logging layer
    let logger = tracing_subscriber::fmt::layer().with_writer(io::stderr);

    // Allow setting RUST_LOG level, or fallback to some level
    let fallback = "info";
    let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .or_else(|_| tracing_subscriber::EnvFilter::try_new(fallback))
        .unwrap();

    // Create a collector?
    let subscriber = tracing_subscriber::Registry::default()
        .with(logger) // .with(layer) Requires tracing_subscriber::layer::SubscriberExt
        .with(env_filter);

    // Initialize tracing
    tracing::subscriber::set_global_default(subscriber).expect("initialize tracing subscriber");

    // Read CLI arguments
    let args: Vec<String> = env::args().collect();

    // one day we can do if-let chains, like:
    // if let arg_len = args.len() && arg_len > 2 {
    // so that we can use the variable, as well as specify the condition for the if block
    if args.len() > 2 {
        tracing::warn!("additional arguments supplied and will be ignored");
    }

    // Build the ingester based on the specified file, or STDIN
    let ingester = if let Some(path) = args.get(1) {
        let reader = BufReader::new(File::open(path)?);
        Ingester::try_from(reader)?
    } else {
        let reader = BufReader::new(io::stdin());
        Ingester::try_from(reader)?
    };

    // Build the graph
    let graph = ProgramGraph::from(ingester.features);

    // Output the graph
    println!("{graph:#?}");

    Ok(())
}

```

## Test

```sh
cargo test
```

## Run examples

> **Note**: I've used examples to test small bits of code for building the library. Usually I use examples for showing how to use the library in various ways.

```sh
cargo run --example simple_tree
```

## Generate docs:

> **Note**: Rust docs are awesome. Cargo can compile example code in comments to make sure they are correct.

```sh
cargo doc --no-deps --lib --document-private-items ## add --open to open in a new browser tab
```
