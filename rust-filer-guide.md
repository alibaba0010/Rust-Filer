# Complete Step-by-Step Guide: Building Filer CLI in Rust

## Project Overview

We'll create a command-line file manager called `filer` that provides interactive prompts for file and folder operations with proper validation and error handling.

## Step 1: Project Setup

### 1.1 Create New Rust Project

```bash
cargo new filer
cd filer
```

### 1.2 Update Cargo.toml

```toml
[package]
name = "filer"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <your.email@example.com>"]
description = "Interactive CLI file and folder manager"
license = "MIT"

[[bin]]
name = "filer"
path = "src/main.rs"

[dependencies]
# CLI framework
clap = { version = "4.5", features = ["derive", "cargo"] }

# Interactive prompts
inquire = "0.75"
dialoguer = "0.11"
console = "0.16"

# File operations
fs_extra = "1.3"
walkdir = "2.5"
trash = "5.2.2"
filetime = "0.2.25"

# Path handling
pathdiff = "0.2.3"
glob = "0.3.3"
path-clean = "1.0.1"

# Error handling
anyhow = "1.0.99"
thiserror = "2.0.14"

# Output formatting
colored = "3.0.0"
indicatif = "0.18"
prettytable-rs = "0.10"

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
toml = "0.9.5"

# Logging
log = "0.4.27"
env_logger = "0.11.8"

# Date/Time
chrono = "0.4.41"

# Platform-specific
[target.'cfg(unix)'.dependencies]
nix = { version = "0.27", features = ["fs", "user"] }
libc = "0.2"

[target.'cfg(windows)'.dependencies]
winapi = { version = "0.3", features = ["fileapi", "winnt", "handleapi"] }

[dev-dependencies]
tempfile = "3.10"
assert_cmd = "2.0"
predicates = "3.1"
```

## Step 2: Project Structure

### 2.1 Create Directory Structure

```
filer/
├── Cargo.toml
├── src/
│   ├── main.rs              # Entry point
│   ├── cli.rs               # CLI command definitions
│   ├── commands/            # Command implementations
│   │   ├── mod.rs
│   │   ├── file.rs          # File operations
│   │   ├── folder.rs        # Folder operations
│   │   ├── permissions.rs   # Permission management
│   │   ├── metadata.rs      # Metadata operations
│   │   └── search.rs        # Search functionality
│   ├── operations/          # Core operations
│   │   ├── mod.rs
│   │   ├── create.rs
│   │   ├── delete.rs
│   │   ├── copy.rs
│   │   ├── move_ops.rs
│   │   └── rename.rs
│   ├── utils/              # Utility functions
│   │   ├── mod.rs
│   │   ├── validation.rs   # Input validation
│   │   ├── prompts.rs      # Interactive prompts
│   │   ├── display.rs      # Output formatting
│   │   └── platform.rs     # Platform-specific code
│   ├── error.rs            # Custom error types
│   └── config.rs           # Configuration
├── tests/
│   └── integration_tests.rs
└── README.md
```

## Step 3: Building and Installing

### 3.1 Build the Project

```bash
# Navigate to project directory
cd filer

# Build in release mode for better performance
cargo build --release
```

### 3.2 Install Globally (Method 1: Cargo Install)

```bash
# Install from the project directory
cargo install --path .

# Or if you want to reinstall/update
cargo install --path . --force
```

### 3.3 Install Globally (Method 2: Manual Copy)

```bash
# On Unix/Linux/macOS
sudo cp target/release/filer /usr/local/bin/

# Make sure it's executable
sudo chmod +x /usr/local/bin/filer

# On Windows (PowerShell as Administrator)
Copy-Item target\release\filer.exe C:\Windows\System32\
```

### 3.4 Add to PATH (if not using cargo install)

```bash
# On Unix/Linux/macOS - Add to ~/.bashrc or ~/.zshrc
export PATH="$PATH:/path/to/filer/target/release"

# On Windows - Add to System Environment Variables
# Go to System Properties → Environment Variables → Path → New
# Add: C:\path\to\filer\target\release
```

### 3.5 Verify Installation

```bash
# Check if filer is available
filer --version

# View help
filer --help
```

## Step 4: Core Implementation Files

### 3.1 src/main.rs

```rust
mod cli;
mod commands;
mod operations;
mod utils;
mod error;
mod config;

use anyhow::Result;
use clap::Parser;
use colored::*;
use env_logger::Env;

fn main() -> Result<()> {
    // Initialize logger
    env_logger::Builder::from_env(Env::default().default_filter_or("info")).init();

    // Parse CLI arguments
    let cli = cli::Cli::parse();

    // Execute command
    match commands::execute(cli.command) {
        Ok(_) => {
            println!("{}", "✓ Operation completed successfully".green());
            Ok(())
        }
        Err(e) => {
            eprintln!("{} {}", "✗ Error:".red(), e);
            std::process::exit(1);
        }
    }
}
```

### 4.2 src/cli.rs

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(
    name = "filer",
    version,
    author,
    about = "Interactive CLI file and folder manager",
    long_about = None
)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    // File operations
    #[command(about = "Create a new file interactively")]
    CreateFile,

    #[command(about = "Delete a file interactively")]
    DeleteFile,

    #[command(about = "Copy a file")]
    CopyFile,

    #[command(about = "Move a file")]
    MoveFile,

    #[command(about = "Rename a file")]
    RenameFile,

    // Folder operations
    #[command(about = "Create a new folder")]
    CreateFolder,

    #[command(about = "Delete a folder")]
    DeleteFolder,

    #[command(about = "Copy a folder")]
    CopyFolder,

    #[command(about = "Move a folder")]
    MoveFolder,

    #[command(about = "List folder contents")]
    List {
        #[arg(default_value = ".")]
        path: String,
    },

    // Metadata operations
    #[command(about = "Show file/folder metadata")]
    Info {
        path: String,
    },

    #[command(about = "Change file/folder permissions")]
    Chmod,

    #[command(about = "Change file/folder ownership")]
    Chown,

    // Search operations
    #[command(about = "Search for files/folders")]
    Search {
        pattern: String,
    },

    #[command(about = "Find files by size")]
    FindBySize,

    // Utility operations
    #[command(about = "Move file/folder to trash")]
    Trash,

    #[command(about = "Calculate directory size")]
    Size {
        path: String,
    },

    #[command(about = "Watch for file changes")]
    Watch {
        path: String,
    },
}
```

### 4.3 src/error.rs

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum FilerError {
    #[error("File not found: {0}")]
    FileNotFound(String),

    #[error("File already exists: {0}")]
    FileExists(String),

    #[error("Invalid file name: {0}")]
    InvalidFileName(String),

    #[error("Invalid file extension: {0}")]
    InvalidExtension(String),

    #[error("Permission denied: {0}")]
    PermissionDenied(String),

    #[error("Invalid path: {0}")]
    InvalidPath(String),

    #[error("Operation cancelled by user")]
    OperationCancelled,

    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),

    #[error("Unknown error: {0}")]
    Unknown(String),
}

pub type Result<T> = std::result::Result<T, FilerError>;
```

### 4.4 src/utils/validation.rs

```rust
use crate::error::{FilerError, Result};
use std::path::Path;

pub fn validate_filename(name: &str) -> Result<()> {
    if name.is_empty() {
        return Err(FilerError::InvalidFileName("Name cannot be empty".into()));
    }

    // Check for invalid characters
    let invalid_chars = ['/', '\\', ':', '*', '?', '"', '<', '>', '|', '\0'];
    for ch in invalid_chars {
        if name.contains(ch) {
            return Err(FilerError::InvalidFileName(
                format!("Name contains invalid character: '{}'", ch)
            ));
        }
    }

    // Check for reserved names on Windows
    #[cfg(windows)]
    {
        let reserved = ["CON", "PRN", "AUX", "NUL", "COM1", "COM2", "COM3", "COM4",
                       "COM5", "COM6", "COM7", "COM8", "COM9", "LPT1", "LPT2",
                       "LPT3", "LPT4", "LPT5", "LPT6", "LPT7", "LPT8", "LPT9"];
        let name_upper = name.to_uppercase();
        for reserved_name in reserved {
            if name_upper == reserved_name || name_upper.starts_with(&format!("{}.", reserved_name)) {
                return Err(FilerError::InvalidFileName(
                    format!("'{}' is a reserved name on Windows", name)
                ));
            }
        }
    }

    Ok(())
}

pub fn validate_extension(ext: &str) -> Result<()> {
    if ext.is_empty() {
        return Err(FilerError::InvalidExtension("Extension cannot be empty".into()));
    }

    if ext.contains('.') {
        return Err(FilerError::InvalidExtension("Extension should not contain '.'".into()));
    }

    // Validate extension contains only alphanumeric characters
    if !ext.chars().all(|c| c.is_alphanumeric() || c == '_') {
        return Err(FilerError::InvalidExtension(
            "Extension should only contain letters, numbers, and underscores".into()
        ));
    }

    Ok(())
}

pub fn validate_path(path: &str) -> Result<()> {
    if path.is_empty() {
        return Err(FilerError::InvalidPath("Path cannot be empty".into()));
    }

    // Check for path traversal attempts
    if path.contains("..") {
        return Err(FilerError::InvalidPath(
            "Path traversal (..) is not allowed for security reasons".into()
        ));
    }

    Ok(())
}

pub fn has_extension(filename: &str) -> bool {
    Path::new(filename)
        .extension()
        .is_some()
}

pub fn add_extension(filename: &str, extension: &str) -> String {
    format!("{}.{}", filename, extension)
}
```

### 4.5 src/utils/prompts.rs

```rust
use crate::error::{FilerError, Result};
use crate::utils::validation;
use inquire::{Text, Confirm, Select, CustomType};
use colored::*;
use std::path::PathBuf;

pub fn prompt_filename() -> Result<String> {
    println!("{}", "📝 File Creation Wizard".blue().bold());
    println!("{}", "─".repeat(40).blue());

    let filename = Text::new("Enter file name:")
        .with_help_message("Type the name for your file (e.g., 'document' or 'document.txt')")
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)?;

    validation::validate_filename(&filename)?;

    if !validation::has_extension(&filename) {
        println!("{}", "⚠ No file extension detected".yellow());

        let add_ext = Confirm::new("Would you like to add a file extension?")
            .with_default(true)
            .prompt()
            .map_err(|_| FilerError::OperationCancelled)?;

        if add_ext {
            let extension = prompt_extension()?;
            Ok(validation::add_extension(&filename, &extension))
        } else {
            Ok(filename)
        }
    } else {
        Ok(filename)
    }
}

pub fn prompt_extension() -> Result<String> {
    let common_extensions = vec![
        "txt", "md", "json", "xml", "csv", "log",
        "rs", "py", "js", "ts", "go", "java",
        "html", "css", "scss", "yaml", "toml",
        "pdf", "doc", "docx", "xls", "xlsx",
        "jpg", "png", "gif", "svg", "mp4", "mp3",
        "Custom..."
    ];

    let selection = Select::new("Select file extension:", common_extensions)
        .with_help_message("Choose from common extensions or enter custom")
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)?;

    let extension = if selection == "Custom..." {
        Text::new("Enter custom extension:")
            .with_help_message("Without the dot (e.g., 'conf', 'config')")
            .prompt()
            .map_err(|_| FilerError::OperationCancelled)?
    } else {
        selection.to_string()
    };

    validation::validate_extension(&extension)?;
    Ok(extension)
}

pub fn prompt_folder_name() -> Result<String> {
    println!("{}", "📁 Folder Creation Wizard".blue().bold());
    println!("{}", "─".repeat(40).blue());

    let folder_name = Text::new("Enter folder name:")
        .with_help_message("Type the name for your folder")
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)?;

    validation::validate_filename(&folder_name)?;
    Ok(folder_name)
}

pub fn prompt_path(message: &str) -> Result<PathBuf> {
    let path_str = Text::new(message)
        .with_help_message("Enter the full or relative path")
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)?;

    validation::validate_path(&path_str)?;
    Ok(PathBuf::from(path_str))
}

pub fn confirm_operation(message: &str) -> Result<bool> {
    Confirm::new(message)
        .with_default(false)
        .with_help_message("This operation cannot be undone")
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)
}

pub fn prompt_overwrite(path: &str) -> Result<bool> {
    println!("{} {}", "⚠".yellow(), format!("'{}' already exists", path).yellow());

    let options = vec!["Overwrite", "Rename", "Skip", "Cancel"];
    let choice = Select::new("What would you like to do?", options)
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)?;

    match choice {
        "Overwrite" => Ok(true),
        "Skip" => Ok(false),
        "Cancel" => Err(FilerError::OperationCancelled),
        "Rename" => {
            let new_name = Text::new("Enter new name:")
                .prompt()
                .map_err(|_| FilerError::OperationCancelled)?;
            validation::validate_filename(&new_name)?;
            // Handle rename logic here
            Ok(false)
        }
        _ => Ok(false),
    }
}

#[cfg(unix)]
pub fn prompt_permissions() -> Result<u32> {
    use inquire::CustomType;

    println!("{}", "🔒 Permission Settings".blue().bold());

    let mode = CustomType::<u32>::new("Enter permissions (octal, e.g., 755):")
        .with_formatter(&|i| format!("{:o}", i))
        .with_error_message("Please enter a valid octal number")
        .with_help_message("Format: owner|group|others (r=4, w=2, x=1)")
        .prompt()
        .map_err(|_| FilerError::OperationCancelled)?;

    Ok(mode)
}
```

### 4.6 src/commands/file.rs

```rust
use crate::error::Result;
use crate::utils::{prompts, validation, display};
use crate::operations;
use colored::*;
use std::fs;
use std::path::Path;

pub fn create_file() -> Result<()> {
    let filename = prompts::prompt_filename()?;
    let path = Path::new(&filename);

    // Check if file already exists
    if path.exists() {
        let overwrite = prompts::prompt_overwrite(&filename)?;
        if !overwrite {
            println!("{}", "Operation cancelled".yellow());
            return Ok(());
        }
    }

    // Ask for initial content
    let add_content = prompts::confirm_operation("Would you like to add initial content?")?;

    let content = if add_content {
        inquire::Editor::new("Enter file content:")
            .with_help_message("Press Ctrl+S to save, Ctrl+Q to quit editor")
            .prompt()
            .unwrap_or_default()
    } else {
        String::new()
    };

    // Create the file
    operations::create::create_file_with_content(&filename, &content)?;

    println!("{} File '{}' created successfully", "✓".green(), filename.green());

    // Show file info
    display::show_file_info(&filename)?;

    Ok(())
}

pub fn delete_file() -> Result<()> {
    let path = prompts::prompt_path("Enter file path to delete:")?;

    if !path.exists() {
        return Err(FilerError::FileNotFound(path.display().to_string()));
    }

    if !path.is_file() {
        return Err(FilerError::InvalidPath("Path is not a file".into()));
    }

    // Show file info before deletion
    display::show_file_info(path.to_str().unwrap())?;

    // Confirm deletion
    let use_trash = prompts::confirm_operation("Move to trash instead of permanent deletion?")?;

    if use_trash {
        operations::delete::move_to_trash(&path)?;
        println!("{} File moved to trash", "✓".green());
    } else {
        let confirm = prompts::confirm_operation(&format!(
            "Are you sure you want to permanently delete '{}'?",
            path.display()
        ))?;

        if confirm {
            operations::delete::delete_file(&path)?;
            println!("{} File deleted permanently", "✓".green());
        } else {
            println!("{}", "Operation cancelled".yellow());
        }
    }

    Ok(())
}

pub fn copy_file() -> Result<()> {
    let source = prompts::prompt_path("Enter source file path:")?;

    if !source.exists() {
        return Err(FilerError::FileNotFound(source.display().to_string()));
    }

    let dest = prompts::prompt_path("Enter destination path:")?;

    // Check if destination exists
    if dest.exists() {
        let overwrite = prompts::prompt_overwrite(&dest.display().to_string())?;
        if !overwrite {
            return Ok(());
        }
    }

    // Show progress bar for large files
    let file_size = fs::metadata(&source)?.len();
    if file_size > 1_000_000 { // 1MB
        operations::copy::copy_file_with_progress(&source, &dest)?;
    } else {
        operations::copy::copy_file(&source, &dest)?;
    }

    println!("{} File copied successfully", "✓".green());
    println!("  {} → {}", source.display(), dest.display().green());

    Ok(())
}

pub fn move_file() -> Result<()> {
    let source = prompts::prompt_path("Enter source file path:")?;

    if !source.exists() {
        return Err(FilerError::FileNotFound(source.display().to_string()));
    }

    let dest = prompts::prompt_path("Enter destination path:")?;

    if dest.exists() {
        let overwrite = prompts::prompt_overwrite(&dest.display().to_string())?;
        if !overwrite {
            return Ok(());
        }
    }

    operations::move_ops::move_file(&source, &dest)?;

    println!("{} File moved successfully", "✓".green());
    println!("  {} → {}", source.display(), dest.display().green());

    Ok(())
}

pub fn rename_file() -> Result<()> {
    let source = prompts::prompt_path("Enter file path to rename:")?;

    if !source.exists() {
        return Err(FilerError::FileNotFound(source.display().to_string()));
    }

    let new_name = prompts::prompt_filename()?;
    let parent = source.parent().unwrap_or(Path::new("."));
    let dest = parent.join(&new_name);

    if dest.exists() {
        let overwrite = prompts::prompt_overwrite(&new_name)?;
        if !overwrite {
            return Ok(());
        }
    }

    operations::rename::rename(&source, &dest)?;

    println!("{} File renamed successfully", "✓".green());
    println!("  {} → {}", source.display(), new_name.green());

    Ok(())
}
```

### 4.7 src/commands/folder.rs

```rust
use crate::error::{Result, FilerError};
use crate::utils::{prompts, display};
use crate::operations;
use colored::*;
use std::path::Path;

pub fn create_folder() -> Result<()> {
    let folder_name = prompts::prompt_folder_name()?;
    let path = Path::new(&folder_name);

    if path.exists() {
        return Err(FilerError::FileExists(folder_name));
    }

    // Ask if should create parent directories
    let create_parents = if folder_name.contains('/') || folder_name.contains('\\') {
        prompts::confirm_operation("Create parent directories if they don't exist?")?
    } else {
        false
    };

    if create_parents {
        operations::create::create_dir_all(&path)?;
    } else {
        operations::create::create_dir(&path)?;
    }

    println!("{} Folder '{}' created successfully", "✓".green(), folder_name.green());

    // Ask if user wants to set permissions (Unix only)
    #[cfg(unix)]
    {
        if prompts::confirm_operation("Would you like to set permissions?")? {
            let mode = prompts::prompt_permissions()?;
            operations::permissions::set_permissions(&path, mode)?;
            println!("{} Permissions set to {:o}", "✓".green(), mode);
        }
    }

    Ok(())
}

pub fn delete_folder() -> Result<()> {
    let path = prompts::prompt_path("Enter folder path to delete:")?;

    if !path.exists() {
        return Err(FilerError::FileNotFound(path.display().to_string()));
    }

    if !path.is_dir() {
        return Err(FilerError::InvalidPath("Path is not a directory".into()));
    }

    // Show folder info
    display::show_folder_info(&path)?;

    // Check if folder is empty
    let is_empty = operations::folder::is_empty(&path)?;

    if !is_empty {
        println!("{}", "⚠ Folder is not empty!".yellow());
        let confirm = prompts::confirm_operation(
            "Delete folder and all its contents? This action cannot be undone!"
        )?;

        if !confirm {
            println!("{}", "Operation cancelled".yellow());
            return Ok(());
        }
    }

    // Ask about trash
    let use_trash = prompts::confirm_operation("Move to trash instead of permanent deletion?")?;

    if use_trash {
        operations::delete::move_to_trash(&path)?;
        println!("{} Folder moved to trash", "✓".green());
    } else {
        if is_empty {
            operations::delete::delete_empty_dir(&path)?;
        } else {
            operations::delete::delete_dir_all(&path)?;
        }
        println!("{} Folder deleted permanently", "✓".green());
    }

    Ok(())
}

pub fn copy_folder() -> Result<()> {
    let source = prompts::prompt_path("Enter source folder path:")?;

    if !source.exists() {
        return Err(FilerError::FileNotFound(source.display().to_string()));
    }

    if !source.is_dir() {
        return Err(FilerError::InvalidPath("Source is not a directory".into()));
    }

    let dest = prompts::prompt_path("Enter destination path:")?;

    if dest.exists() {
        let overwrite = prompts::confirm_operation(
            "Destination exists. Merge folders?"
        )?;
        if !overwrite {
            return Ok(());
        }
    }

    operations::copy::copy_dir_with_progress(&source, &dest)?;

    println!("{} Folder copied successfully", "✓".green());
    println!("  {} → {}", source.display(), dest.display().green());

    Ok(())
}

pub fn move_folder() -> Result<()> {
    let source = prompts::prompt_path("Enter source folder path:")?;

    if !source.exists() {
        return Err(FilerError::FileNotFound(source.display().to_string()));
    }

    let dest = prompts::prompt_path("Enter destination path:")?;

    if dest.exists() {
        return Err(FilerError::FileExists(dest.display().to_string()));
    }

    operations::move_ops::move_dir(&source, &dest)?;

    println!("{} Folder moved successfully", "✓".green());
    println!("  {} → {}", source.display(), dest.display().green());

    Ok(())
}

pub fn list_folder(path: &str) -> Result<()> {
    let path = Path::new(path);

    if !path.exists() {
        return Err(FilerError::FileNotFound(path.display().to_string()));
    }

    if !path.is_dir() {
        return Err(FilerError::InvalidPath("Path is not a directory".into()));
    }

    display::list_directory_contents(&path)?;

    Ok(())
}
```

### 4.8 src/commands/mod.rs

```rust
use crate::error::{Result, FilerError};
use crate::utils::{prompts, display};
use crate::operations;
use colored::*;
use std::path::Path;

pub fn create_folder() -> Result<()> {
    let folder_name = prompts::prompt_folder_name()?;
    let path = Path::new(&folder_name);

    if path.exists() {
        return Err(FilerError::FileExists(folder_name));
    }

    // Ask if should create parent directories
    let create_parents = if folder_name.contains('/') || folder_name.contains('\\') {
        prompts::confirm_operation("Create parent directories if they don't exist?")?
    } else {
        false
    };

    if create_parents {
        operations::create::create_dir_all(&path)?;
    } else {
        operations::create::create_dir(&path)?;
    }

    println!("{} Folder '{}' created successfully", "✓".green(), folder_name.green());

    // Ask if user wants to set permissions (Unix only)
    #[cfg(unix)]
    {
        if prompts::confirm_operation("Would you like to set permissions?")? {
            let mode = prompts::prompt_permissions()?;
            operations::permissions::set_permissions(&path, mode)?;
            println!("{} Permissions set to {:o}", "✓".green(), mode);
        }
    }

    Ok(())
}

pub fn delete_folder() -> Result<()> {
    let path = prompts::prompt_path("Enter folder path to delete:")?;

    if !path.exists() {
        return Err(FilerError::FileNotFound(path.display().to_string()));
    }

    if !path.is_dir() {
        return Err(FilerError::InvalidPath("Path is not a directory".into()));
    }

    // Show folder info
    display::show_folder_info(&path)?;

    // Check if folder is empty
    let is_empty = operations::folder::is_empty(&path)?;

    if !is_empty {
        println!("{}", "⚠ Folder is not empty!".yellow());
        let confirm = prompts::confirm_operation(
            "Delete folder and all its contents? This action cannot be undone!"
        )?;

        if !confirm {
            println!("{}", "Operation cancelled".yellow());
            return Ok(());
        }
    }

    // Ask about trash
    let use_trash = prompts::confirm_operation("Move to trash instead of permanent deletion?")?;

    if use_trash {
        operations::delete::move_to_trash(&path)?;
        println!("{} Folder moved to trash", "✓".green());
    } else {
        if is_empty {
            operations::delete::delete_empty_dir(&path)?;
        } else {
            operations::delete::delete_dir_all(&path)?;
        }
        println!("{} Folder deleted permanently", "✓".green());
    }

    Ok(())
}

pub fn list_folder(path: &str) -> Result<()> {
    let path = Path::new(path);

    if !path.exists() {
        return Err(FilerError::FileNotFound(path.display().to_string()));
    }

    if !path.is_dir() {
        return Err(FilerError::InvalidPath("Path is not a directory".into()));
    }

    display::list_directory_contents(&path)?;

    Ok(())
}
```

### 4.8 src/commands/mod.rs

```rust
mod file;
mod folder;
mod permissions;
mod metadata;
mod search;

use crate::cli::Commands;
use crate::error::Result;

pub fn execute(command: Commands) -> Result<()> {
    match command {
        // File operations
        Commands::CreateFile => file::create_file(),
        Commands::DeleteFile => file::delete_file(),
        Commands::CopyFile => file::copy_file(),
        Commands::MoveFile => file::move_file(),
        Commands::RenameFile => file::rename_file(),

        // Folder operations
        Commands::CreateFolder => folder::create_folder(),
        Commands::DeleteFolder => folder::delete_folder(),
        Commands::CopyFolder => folder::copy_folder(),
        Commands::MoveFolder => folder::move_folder(),
        Commands::List { path } => folder::list_folder(&path),

        // Metadata operations
        Commands::Info { path } => metadata::show_info(&path),
        Commands::Chmod => permissions::change_permissions(),
        Commands::Chown => permissions::change_ownership(),

        // Search operations
        Commands::Search { pattern } => search::search_files(&pattern),
        Commands::FindBySize => search::find_by_size(),

        // Utility operations
        Commands::Trash => file::move_to_trash_interactive(),
        Commands::Size { path } => metadata::calculate_size(&path),
        Commands::Watch { path } => metadata::watch_changes(&path),
    }
}
```

### 4.9 src/operations/create.rs

```rust
use crate::error::Result;
use std::fs::{self, File, OpenOptions};
use std::io::Write;
use std::path::Path;

pub fn create_file_with_content(path: &str, content: &str) -> Result<()> {
    let mut file = File::create(path)?;
    file.write_all(content.as_bytes())?;
    file.sync_all()?;
    Ok(())
}

pub fn create_file(path: &Path) -> Result<()> {
    File::create(path)?;
    Ok(())
}

pub fn create_dir(path: &Path) -> Result<()> {
    fs::create_dir(path)?;
    Ok(())
}

pub fn create_dir_all(path: &Path) -> Result<()> {
    fs::create_dir_all(path)?;
    Ok(())
}

pub fn create_with_permissions(path: &Path, mode: u32) -> Result<()> {
    let file = File::create(path)?;

    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        let permissions = fs::Permissions::from_mode(mode);
        fs::set_permissions(path, permissions)?;
    }

    Ok(())
}
```

### 4.10 src/utils/display.rs

```rust
use crate::error::Result;
use chrono::{DateTime, Local};
use colored::*;
use prettytable::{Table, Row, Cell, format};
use std::fs;
use std::path::Path;
use walkdir::WalkDir;

pub fn show_file_info(path: &str) -> Result<()> {
    let path = Path::new(path);
    let metadata = fs::metadata(path)?;

    println!("\n{}", "📄 File Information".blue().bold());
    println!("{}", "─".repeat(50).blue());

    let mut table = Table::new();
    table.set_format(*format::consts::FORMAT_CLEAN);

    table.add_row(Row::new(vec![
        Cell::new("Name:").style_spec("bF"),
        Cell::new(&path.file_name().unwrap().to_string_lossy()),
    ]));

    // Count items in folder
    let (file_count, dir_count, total_size) = count_folder_contents(path)?;

    table.add_row(Row::new(vec![
        Cell::new("Files:").style_spec("bF"),
        Cell::new(&file_count.to_string()),
    ]));

    table.add_row(Row::new(vec![
        Cell::new("Folders:").style_spec("bF"),
        Cell::new(&dir_count.to_string()),
    ]));

    table.add_row(Row::new(vec![
        Cell::new("Total Size:").style_spec("bF"),
        Cell::new(&format_size(total_size)),
    ]));

    if let Ok(modified) = metadata.modified() {
        let datetime: DateTime<Local> = modified.into();
        table.add_row(Row::new(vec![
            Cell::new("Modified:").style_spec("bF"),
            Cell::new(&datetime.format("%Y-%m-%d %H:%M:%S").to_string()),
        ]));
    }

    table.printstd();
    println!();

    Ok(())
}

pub fn list_directory_contents(path: &Path) -> Result<()> {
    println!("\n{} {}", "📁".blue(), path.display().to_string().bold());
    println!("{}", "─".repeat(70).blue());

    let mut table = Table::new();
    table.set_format(*format::consts::FORMAT_CLEAN);

    // Header
    table.add_row(Row::new(vec![
        Cell::new("Type").style_spec("bFc"),
        Cell::new("Name").style_spec("bFc"),
        Cell::new("Size").style_spec("bFc"),
        Cell::new("Modified").style_spec("bFc"),
    ]));

    let mut entries: Vec<_> = fs::read_dir(path)?
        .filter_map(|e| e.ok())
        .collect();

    // Sort entries: directories first, then files
    entries.sort_by_key(|entry| {
        let is_dir = entry.file_type().map(|ft| ft.is_dir()).unwrap_or(false);
        (!is_dir, entry.file_name())
    });

    for entry in entries {
        let metadata = entry.metadata()?;
        let file_type = if metadata.is_dir() { "📁" } else { "📄" };
        let name = entry.file_name();
        let size = if metadata.is_dir() {
            "<DIR>".to_string()
        } else {
            format_size(metadata.len())
        };

        let modified = if let Ok(time) = metadata.modified() {
            let datetime: DateTime<Local> = time.into();
            datetime.format("%Y-%m-%d %H:%M").to_string()
        } else {
            "Unknown".to_string()
        };

        table.add_row(Row::new(vec![
            Cell::new(file_type),
            Cell::new(&name.to_string_lossy()),
            Cell::new(&size),
            Cell::new(&modified),
        ]));
    }

    table.printstd();
    println!();

    Ok(())
}

fn format_size(size: u64) -> String {
    const UNITS: &[&str] = &["B", "KB", "MB", "GB", "TB"];
    let mut size = size as f64;
    let mut unit_index = 0;

    while size >= 1024.0 && unit_index < UNITS.len() - 1 {
        size /= 1024.0;
        unit_index += 1;
    }

    if unit_index == 0 {
        format!("{} {}", size as u64, UNITS[unit_index])
    } else {
        format!("{:.2} {}", size, UNITS[unit_index])
    }
}

#[cfg(unix)]
fn format_permissions(mode: u32) -> String {
    let user = triplet((mode >> 6) & 0o7);
    let group = triplet((mode >> 3) & 0o7);
    let other = triplet(mode & 0o7);
    format!("{}{}{}", user, group, other)
}

#[cfg(unix)]
fn triplet(mode: u32) -> String {
    let r = if mode & 0o4 != 0 { "r" } else { "-" };
    let w = if mode & 0o2 != 0 { "w" } else { "-" };
    let x = if mode & 0o1 != 0 { "x" } else { "-" };
    format!("{}{}{}", r, w, x)
}

fn count_folder_contents(path: &Path) -> Result<(usize, usize, u64)> {
    let mut file_count = 0;
    let mut dir_count = 0;
    let mut total_size = 0;

    for entry in WalkDir::new(path).max_depth(1).skip(1) {
        let entry = entry?;
        let metadata = entry.metadata()?;

        if metadata.is_file() {
            file_count += 1;
            total_size += metadata.len();
        } else if metadata.is_dir() {
            dir_count += 1;
        }
    }

    Ok((file_count, dir_count, total_size))
} = Table::new();
    table.set_format(*format::consts::FORMAT_CLEAN);

    table.add_row(Row::new(vec![
        Cell::new("Name:").style_spec("bF"),
        Cell::new(&path.file_name().unwrap().to_string_lossy()),
    ]));

    table.add_row(Row::new(vec![
        Cell::new("Size:").style_spec("bF"),
        Cell::new(&format_size(metadata.len())),
    ]));

    table.add_row(Row::new(vec![
        Cell::new("Type:").style_spec("bF"),
        Cell::new(if metadata.is_file() { "File" }
                 else if metadata.is_dir() { "Directory" }
                 else { "Other" }),
    ]));

    if let Ok(modified) = metadata.modified() {
        let datetime: DateTime<Local> = modified.into();
        table.add_row(Row::new(vec![
            Cell::new("Modified:").style_spec("bF"),
            Cell::new(&datetime.format("%Y-%m-%d %H:%M:%S").to_string()),
        ]));
    }

    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        let mode = metadata.permissions().mode();
        table.add_row(Row::new(vec![
            Cell::new("Permissions:").style_spec("bF"),
            Cell::new(&format!("{:o} ({})", mode & 0o777, format_permissions(mode))),
        ]));
    }

    table.printstd();
    println!();

    Ok(())
}

pub fn show_folder_info(path: &Path) -> Result<()> {
    let metadata = fs::metadata(path)?;

    println!("\n{}", "📁 Folder Information".blue().bold());
    println!("{}", "─".repeat(50).blue());

    let mut table
```
