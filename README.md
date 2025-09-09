use std::fs::{self, OpenOptions};
use std::io::{self, BufRead, BufReader, Write};
use std::path::Path;
use std::{thread, time::Duration};

use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "Rust CLI Toolbox")]
#[command(about = "A multi-tool CLI written in Rust", long_about = None)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Manage your to-do list
    Todo {
        #[arg(short, long)]
        add: Option<String>,

        #[arg(short, long)]
        list: bool,
    },

    /// Perform basic calculations
    Calc {
        a: f64,
        operator: String,
        b: f64,
    },

    /// Convert units (e.g. temperature, distance)
    Convert {
        #[arg(short, long)]
        from: String,

        #[arg(short, long)]
        to: String,

        #[arg(short, long)]
        value: f64,
    },

    /// Start a timer (in minutes)
    Timer {
        #[arg(short, long)]
        minutes: u64,
    },

    /// Rename files in a directory
    Rename {
        #[arg(short, long)]
        directory: String,

        #[arg(short, long)]
        prefix: String,
    },
}

fn main() {
    let cli = Cli::parse();

    match &cli.command {
        Commands::Todo { add, list } => run_todo(add, *list),
        Commands::Calc { a, operator, b } => run_calc(*a, operator, *b),
        Commands::Convert { from, to, value } => run_convert(from, to, *value),
        Commands::Timer { minutes } => run_timer(*minutes),
        Commands::Rename { directory, prefix } => run_rename(directory, prefix),
    }
}

// ------------------ To-Do Tool ------------------
fn run_todo(add: &Option<String>, list: bool) {
    let path = "todo.txt";

    if let Some(task) = add {
        let mut file = OpenOptions::new()
            .append(true)
            .create(true)
            .open(path)
            .expect("Failed to open file.");
        writeln!(file, "{}", task).expect("Failed to write to file.");
        println!("✅ Added task: {}", task);
    }

    if list {
        let file = OpenOptions::new().read(true).open(path);
        match file {
            Ok(f) => {
                let reader = BufReader::new(f);
                println!("📝 To-Do List:");
                for (i, line) in reader.lines().enumerate() {
                    println!("{}. {}", i + 1, line.unwrap());
                }
            }
            Err(_) => println!("📭 No tasks found."),
        }
    }
}

// ------------------ Calculator ------------------
fn run_calc(a: f64, op: &str, b: f64) {
    let result = match op {
        "+" => a + b,
        "-" => a - b,
        "*" => a * b,
        "/" => {
            if b == 0.0 {
                println!("❌ Cannot divide by zero.");
                return;
            }
            a / b
        }
        _ => {
            println!("❌ Unsupported operator: {}", op);
            return;
        }
    };

    println!("🧮 Result: {}", result);
}

// ------------------ Unit Converter ------------------
fn run_convert(from: &str, to: &str, value: f64) {
    match (from.to_lowercase().as_str(), to.to_lowercase().as_str()) {
        ("celsius", "fahrenheit") => println!("🌡️ {}°C = {}°F", value, value * 1.8 + 32.0),
        ("fahrenheit", "celsius") => println!("🌡️ {}°F = {}°C", value, (value - 32.0) / 1.8),
        ("km", "miles") => println!("📏 {} km = {} miles", value, value * 0.621371),
        ("miles", "km") => println!("📏 {} miles = {} km", value, value / 0.621371),
        _ => println!("❌ Unsupported conversion from '{}' to '{}'", from, to),
    }
}

// ------------------ Timer ------------------
fn run_timer(minutes: u64) {
    let seconds = minutes * 60;
    println!("⏳ Starting timer for {} minutes...", minutes);
    thread::sleep(Duration::from_secs(seconds));
    println!("⏰ Time's up!");
}

// ------------------ File Renamer ------------------
fn run_rename(directory: &str, prefix: &str) {
    let paths = match fs::read_dir(directory) {
        Ok(paths) => paths,
        Err(_) => {
            println!("❌ Failed to read directory: {}", directory);
            return;
        }
    };

    for (i, file) in paths.filter_map(Result::ok).enumerate() {
        let path = file.path();
        if path.is_file() {
            let ext = path
                .extension()
                .and_then(|e| e.to_str())
                .unwrap_or("");
            let new_name = format!("{}{}_{}.{}", prefix, i + 1, chrono::Local::now().timestamp(), ext);
            let new_path = Path::new(directory).join(new_name);
            match fs::rename(&path, &new_path) {
                Ok(_) => println!(
                    "🔁 Renamed {:?} → {:?}",
                    path.file_name().unwrap(),
                    new_path.file_name().unwrap()
                ),
                Err(e) => println!("❌ Failed to rename file: {}", e),
            }
        }
    }
}
