# 1.1 Welcome to the Blog Aggregator

We're going to build an RSS feed aggregator in Go! We'll call it "Gator", you know, because aggreGATOR 🐊. Anyhow, it's a CLI tool that allows users to:

Add RSS feeds from across the internet to be collected
Store the collected posts in a PostgreSQL database
Follow and unfollow RSS feeds that other users have added
View summaries of the aggregated posts in the terminal, with a link to the full post
RSS feeds are a way for websites to publish updates to their content. You can use this project to keep up with your favorite blogs, news sites, podcasts, and more!

Prerequisites
The project assumes that you're already familiar with the Go programming language and SQL databases.

Learning Goals
Learn how to integrate a Go application with a PostgreSQL database
Practice using your SQL skills to query and migrate a database (using sqlc and goose, two lightweight tools for typesafe SQL in Go)
Learn how to write a long-running service that continuously fetches new posts from RSS feeds and stores them in the database
Assignment
Before we start, let's get you setup. You'll need both the go toolchain (version 1.25+) and the Boot.dev CLI installed. If you don't already have them, here are the installation instructions.

Run and submit the CLI tests.

# 1.2 Config

Gator is a multi-user CLI application. There's no server (other than the database), so it's only intended for local use, but just like games in the 90's and early 2000's, that doesn't mean we can't have multiplayer functionality on a single device!

We'll use a single JSON file to keep track of two things:

Who is currently logged in
The connection credentials for the PostgreSQL database
The JSON file should have this structure (when prettified):

{
  "db_url": "connection_string_goes_here",
  "current_user_name": "username_goes_here"
}

Note: There's no user-based authentication for this app. If someone has the database credentials, they can act as any user. We'll cover auth in other courses. We want to focus on SQL, CLIs and long-running services for this project.

Assignment
Manually create a config file in your home directory, ~/.gatorconfig.json, with the following content:
{
  "db_url": "postgres://example"
}

Don't worry about adding current_user_name, that will be set by the application.

Initialize a new Go module, go mod init. Create a new file, main.go. Add a main function. It doesn't need to do anything yet.
Create the internal and internal/config directories to contain the config internal package. This package will be responsible for reading and writing the JSON file.
This package should have the following functionality exported so the main package can use it:

Export a Config struct that represents the JSON file structure, including struct tags.
Export a Read function that reads the JSON file found at ~/.gatorconfig.json and returns a Config struct. It should read the file from the HOME directory, then decode the JSON string into a new Config struct. I used os.UserHomeDir to get the location of HOME.
Export a SetUser method on the Config struct that writes the config struct to the JSON file after setting the current_user_name field.
I also wrote a few non-exported helper functions and added a constant to hold the filename.

getConfigFilePath() (string, error)
write(cfg Config) error
const configFileName = ".gatorconfig.json"
But you can implement the internals of the package however you like.

Update the main function to:
Read the config file.
Set the current user to "lane" (actually, you should use your name instead) and update the config file on disk.
Read the config file again and print the contents of the config struct to the terminal.
Run and submit the CLI tests from the root of your project.


# 1.3 Commands

Gator is a CLI application, and like many CLI applications, it has a set of valid commands. For example:

gator login - sets the current user in the config
gator register - adds a new user to the database
gator users - lists all the users in the database
etc.
We'll be hand-rolling our CLI rather than using a framework like Cobra or Bubble Tea. This will give us a better understanding of how CLI applications work.

Assignment
Let's start with a simple login command. For now, all it will do is set the current user in the config file. Usage:

gator login <username>

We want to add many commands to our CLI, so let's build a flexible system that will allow us to register new commands easily.

Before we can worry about command handlers, we need to think about how we will give our handlers access to the application state (later the database connection, but, for now, the config file). Create a state struct that holds a pointer to a config.
Create a command struct. A command contains a name and a slice of string arguments. For example, in the case of the login command, the name would be "login" and the handler will expect the arguments slice to contain one string, the username.
Create a login handler function: func handlerLogin(s *state, cmd command) error. This will be the function signature of all command handlers.
If the command's arg's slice is empty, return an error; the login handler expects a single argument, the username.
Use the state's access to the config struct to set the user to the given username. Remember to return any errors.
Print a message to the terminal that the user has been set.
Create a commands struct. This will hold all the commands the CLI can handle. Add a map[string]func(*state, command) error field to it. This will be a map of command names to their handler functions.
Implement the following methods on the commands struct:
func (c *commands) run(s*state, cmd command) error - This method runs a given command with the provided state if it exists.
func (c *commands) register(name string, f func(*state, command) error) - This method registers a new handler function for a command name.
In the main function, remove the manual update of the config file. Instead, simply read the config file, and store the config in a new instance of the state struct.
Create a new instance of the commands struct with an initialized map of handler functions.
Register a handler function for the login command.
Use os.Args to get the command-line arguments passed in by the user.
If there are fewer than 2 arguments, print an error message to the terminal and exit. Why two? The first argument is automatically the program name, which we ignore, and we require a command name.

You'll need to split the command-line arguments into the command name and the arguments slice to create a command instance. Use the commands.run method to run the given command and print any errors returned.
Test your CLI. Here are a few cases to consider:
go run . - Exit with code 1 and print an error that not enough arguments were provided.
go run . login - Exit with code 1 and print an error that a username is required.
go run . login alice - should set the current user in the config file to "alice" (and not overwrite the DB string)
Run and submit the CLI tests from the root of your project.

# 2.1 Postgres

PostgreSQL is a production-ready, open-source database. It's a great choice for many web applications, and as a back-end engineer, it might be the single most important database to be familiar with.

How Does PostgreSQL Work?
Postgres, like most other database technologies, is itself a server. It listens for requests on a port (Postgres' default is :5432), and responds to those requests. To interact with Postgres, first you will install the server and start it. Then, you can connect to it using a client like psql or PGAdmin.

Install Postgres v15 or later.
macOS with brew

brew install postgresql@15

Linux / WSL (Debian). Here are the docs from Microsoft, but simply:

sudo apt update
sudo apt install postgresql postgresql-contrib

Ensure the installation worked. The psql command-line utility is the default client for Postgres. Use it to make sure you're on version 15+ of Postgres:
psql --version

(Linux only) Update postgres password:
sudo passwd postgres

Enter a password, and be sure you won't forget it. You can just use something easy like postgres.

Start the Postgres server in the background
Mac: brew services start postgresql@15
Linux: sudo service postgresql start
Connect to the server. I recommend simply using the psql client. It's the "default" client for Postgres, and it's a great way to interact with the database. While it's not as user-friendly as a GUI like PGAdmin, it's a great tool to be able to do at least basic operations with.
Enter the psql shell:

Mac: psql postgres
Linux: sudo -u postgres psql
You should see a new prompt that looks like this:

postgres=#

Create a new database. I called mine gator:
CREATE DATABASE gator;

Connect to the new database:
\c gator

You should see a new prompt that looks like this:

gator=#

Set the user password (Linux only)
ALTER USER postgres PASSWORD 'postgres';

For simplicity, I used postgres as the password. Before, we altered the system user's password, now we're altering the database user's password.

Query the database
From here you can run SQL queries against the gator database. For example, to see the version of Postgres you're running, you can run:

SELECT version();

You can type exit to leave the psql shell.

**important**

psql "postgres://carlosinfante:@localhost:5432/gator"

goose postgres "postgres://carlosinfante:@localhost:5432/gator" up


