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

# 2.2 Goose Migrations

Goose is a database migration tool written in Go. It runs migrations from a set of SQL files, making it a perfect fit for this project (we wanna stay close to the raw SQL).

What Is a Migration?
A migration is just a set of changes to your database table. You can have as many migrations as needed as your requirements change over time. For example, one migration might create a new table, one might delete a column, and one might add 2 new columns.

An "up" migration moves the state of the database from its current schema to the schema that you want. So, to get a "blank" database to the state it needs to be ready to run your application, you run all the "up" migrations.

If something breaks, you can run one of the "down" migrations to revert the database to a previous state. "Down" migrations are also used if you need to reset a local testing database to a known state.

Assignment
Install Goose
Goose is just a command line tool that happens to be written in Go. I recommend installing it using go install:

go install github.com/pressly/goose/v3/cmd/goose@latest

Run goose -version to make sure it's installed correctly.

Create a users migration in a new sql/schema directory.
A "migration" in Goose is just a .sql file with some SQL queries and some special comments. Our first migration should just create a users table. The simplest format for these files is:

number_name.sql

For example, I created a file in sql/schema called 001_users.sql with the following contents:

-- +goose Up
CREATE TABLE ...

-- +goose Down
DROP TABLE users;

Write out the CREATE TABLE statement in full, I left it blank for you to fill in. A user should have 4 fields:

id: a UUID that will serve as the primary key
created_at: a TIMESTAMP that can not be null
updated_at: a TIMESTAMP that can not be null
name: a unique string that can not be null
The -- +goose Up and -- +goose Down comments are case sensitive and required. They tell Goose how to run the migration in each direction.

Get your connection string. A connection string is just a URL with all of the information needed to connect to a database. The format is:
protocol://username:password@host:port/database

Here are examples:

macOS (no password, your username): postgres://wagslane:@localhost:5432/gator
Linux (password from last lesson, postgres user): postgres://postgres:postgres@localhost:5432/gator
Test your connection string by running psql, for example:

psql "postgres://wagslane:@localhost:5432/gator"

It should connect you to the gator database directly. If it's working, great. exit out of psql and save the connection string.

Run the up migration.
cd into the sql/schema directory and run:

goose postgres <connection_string> up

example

goose postgres "postgres://wagslane:@localhost:5432/gator" up

Run your migration! Make sure it works by using psql to find your newly created users table:

psql gator
\dt

Run the down migration to make sure it works (it should just drop the table). When you're satisfied, run the up migration again to recreate the table.
Add the connection string to the .gatorconfig.json file instead of the example string we used earlier. When using it with goose, you'll use it in the format we just used. However, here in the config file it needs an additional sslmode=disable query string:
protocol://username:password@host:port/database?sslmode=disable

Your application code needs to know to not try to use SSL locally.

Run and submit the CLI tests.

# 2.3 SQLC

SQLC is an amazing Go program that generates Go code from SQL queries. It's not exactly an ORM, but rather a tool that makes working with raw SQL easy and type-safe.

We will use Goose to manage our database migrations (the schema), and SQLC to generate Go code that our application can use to interact with the database (run queries).

Assignment
Install SQLC.
SQLC is just a command line tool, it's not a package that we need to import. I recommend installing it using go install. Installing Go CLI tools with go install is easy and ensures compatibility with your Go environment.

go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest

Then run sqlc version to make sure it's installed correctly.

Configure SQLC. You'll always run the sqlc command from the root of your project. Create a file called sqlc.yaml in the root of your project. Here is mine:
version: "2"
sql:

- schema: "sql/schema"
    queries: "sql/queries"
    engine: "postgresql"
    gen:
      go:
        out: "internal/database"

We're telling SQLC to look in the sql/schema directory for our schema structure (which is the same set of files that Goose uses, but sqlc automatically ignores "down" migrations), and in the sql/queries directory for queries. We're also telling it to generate Go code in the internal/database directory.

Write a query to create a user. Inside the sql/queries directory, create a file called users.sql. Here's the format:
-- name: CreateUser :one
INSERT INTO users (id, created_at, updated_at, name)
VALUES (
    $1,
    $2,
    $3,
    $4
)
RETURNING *;

$1, $2, $3, and $4 are parameters that we'll be able to pass into the query in our Go code. The :one at the end of the query name tells SQLC that we expect to get back a single row (the created user).

Keep the SQLC postgres docs handy, you'll probably need to refer to them again later.

Generate the Go code. Run sqlc generate from the root of your project. It should create a new package of go code in internal/database. You'll notice that the generated code relies on Google's uuid package, so you'll need to add that to your module:
go get github.com/google/uuid

Write another query to get a user by name. I used the comment -- name: GetUser :one to tell SQLC that I expect to get back a single row. Again, generate the Go code to ensure that it works.
Import a PostgreSQL driver.
We need to add and import a Postgres driver so our program knows how to talk to the database. Install it in your module:

go get github.com/lib/pq

Add this import to the top of your main.go file:

import _ "github.com/lib/pq"

This is one of my least favorite things working with SQL in Go currently. You have to import the driver, but you don't use it directly anywhere in your code. The underscore tells Go that you're importing it for its side effects, not because you need to use it.
Open a connection to the database, and store it in the state struct:
In main(), load in your database URL to the config struct and sql.Open() a connection to your database:

db, err := sql.Open("postgres", dbURL)

Use your generated database package to create a new *database.Queries, and store it in your state struct:

dbQueries := database.New(db)

type state struct {
 db  *database.Queries
 cfg*config.Config
}

Create a register handler and register it with the commands. Usage:
go run . register lane

It should:

Ensure that a name was passed in the args.
Create a new user in the database. It should have access to the CreateUser query through the state -> db struct.
Pass context.Background() to the query to create an empty Context argument.
Use the uuid.New() function to generate a new UUID for the user.
created_at and updated_at should be the current time.
Use the provided name.
Exit with code 1 if a user with that name already exists.
Set the current user in the config to the given name.
Print a message that the user was created, and log the user's data to the console for your own debugging.
Test the register command by running it with a name:

go run . register lane

Use psql to verify that the user was created:

Mac: psql postgres
Linux: sudo -u postgres psql
\c gator

SELECT * FROM users;

Update the login command handler to error (and exit with code 1) if the given username doesn't exist in the database. You can't login to an account that doesn't exist!
Take a good look at the tests and run them before submitting.

Be sure to migrate down and back up with goose before each run/submit, because the tests assume a clean database.


# 2.4 Reset

When doing end-to-end testing on a system like this, it's always a bit annoying to have to manually reset the state of the database. For example, the register command fails if you run it again with the same username because the user already exists.

That's the behavior we want, but we have to manually reset the database each time we run tests.

Assignment
Let's add a command to our CLI that will reset the database to a blank state. This will make development much easier for us (although probably not useful in production... maybe even disastrous).

Add a new query to the database that deletes all the users in the users table. Generate the Go code with sqlc generate. Use the :exec tag in the comment.
Add a new command called reset that calls the query. Report back to the user about whether or not it was successful with an appropriate exit code.

# 2.5 Reset

When doing end-to-end testing on a system like this, it's always a bit annoying to have to manually reset the state of the database. For example, the register command fails if you run it again with the same username because the user already exists.

That's the behavior we want, but we have to manually reset the database each time we run tests.

Assignment
Let's add a command to our CLI that will reset the database to a blank state. This will make development much easier for us (although probably not useful in production... maybe even disastrous).

Add a new query to the database that deletes all the users in the users table. Generate the Go code with sqlc generate. Use the :exec tag in the comment.
Add a new command called reset that calls the query. Report back to the user about whether or not it was successful with an appropriate exit code.

# 3.1 RSS

The whole point of the gator program is to fetch the RSS feed of a website and store its content in a structured format in our database. That way we can display it nicely in our CLI.

RSS stands for "Really Simple Syndication" and is a way to get the latest content from a website in a structured format. It's fairly ubiquitous on the web: most content sites have an RSS feed.

Structure of an RSS Feed
RSS is a specific structure of XML (I know, gross). We will keep it simple and only worry about a few fields. Here's an example of the documents we'll parse:

<rss xmlns:atom="http://www.w3.org/2005/Atom" version="2.0">
<channel>
  <title>RSS Feed Example</title>
  <link>https://www.example.com</link>
  <description>This is an example RSS feed</description>
  <item>
    <title>First Article</title>
    <link>https://www.example.com/article1</link>
    <description>This is the content of the first article.</description>
    <pubDate>Mon, 06 Sep 2021 12:00:00 GMT</pubDate>
  </item>
  <item>
    <title>Second Article</title>
    <link>https://www.example.com/article2</link>
    <description>Here's the content of the second article.</description>
    <pubDate>Tue, 07 Sep 2021 14:30:00 GMT</pubDate>
  </item>
</channel>
</rss>

We'll then directly unmarshal this kind of document into structs like this:

type RSSFeed struct {
 Channel struct {
  Title       string    `xml:"title"`
  Link        string    `xml:"link"`
  Description string    `xml:"description"`
  Item        []RSSItem `xml:"item"`
 } `xml:"channel"`
}

type RSSItem struct {
 Title       string `xml:"title"`
 Link        string `xml:"link"`
 Description string `xml:"description"`
 PubDate     string `xml:"pubDate"`
}

If there are any extra fields in the XML, the parser will just discard them, and if any are missing, the parser will leave them as their zero value.

Assignment
Write a func fetchFeed(ctx context.Context, feedURL string) (*RSSFeed, error) function. It should fetch a feed from the given URL, and, assuming that nothing goes wrong, return a filled-out RSSFeed struct. Here are some useful docs (be sure to check the Overviews for examples if the entry lacks any):
http.NewRequestWithContext
http.Client.Do
I set the User-Agent header to gator in the request with request.Header.Set. This is a common practice to identify your program to the server.
io.ReadAll
xml.Unmarshal (works the same as json.Unmarshal)
Use the html.UnescapeString function to decode escaped HTML entities (like &ldquo;). You'll need to run the Title and Description fields (of both the entire channel as well as the items) through this function.
Add an agg command. Later this will be our long-running aggregator service. For now, we'll just use it to fetch a single feed and ensure our parsing works. It should fetch the feed found at <https://www.wagslane.dev/index.xml> and print the entire struct to the console.


# 3.2 Feeds

Now that we have a way to fetch feeds from the internet, we need to store them in our database.

Assignment
Create a feeds table.
Like any table in our DB, we'll need the standard id, created_at, and updated_at fields. We'll also need a few more:

name: The name of the feed (like "The Changelog, or "The Boot.dev Blog")
url: The URL of the feed
user_id: The ID of the user who added this feed
Make the url field unique so that in the future we aren't downloading duplicate posts.

Use an ON DELETE CASCADE constraint on the user_id foreign key so that if a user is deleted, all of their feeds are automatically deleted as well. This will ensure we have no orphaned records and that deleting the users in the reset command also deletes all of their feeds.

Write the appropriate migrations and run them.

Add a new query to create a feed, then use sqlc generate to generate the Go code.
Add a new command called addfeed. It takes two args:
name: The name of the feed
url: The URL of the feed
At the top of the handler, get the current user from the database and connect the feed to that user.

If everything goes well, print out the fields of the new feed record.

# 3.3 List Feeds

We need a way to inspect all the feeds in the database.

Assignment
Add a new feeds handler. It takes no arguments and prints all the feeds in the database to the console. Be sure to include:

The name of the feed
The URL of the feed
The name of the user that created the feed (you might need a new SQL query)

# 4.1 Follow

Feeds currently have a creator (the user_id on the feed record) but there's no way for other users to follow feeds created by other users. And because we're making feeds unique by URL, they can't even add a feed that another user has already added.

We need to introduce a many-to-many relationship between users and feeds. Many users can follow the same feed, and a user can follow many feeds.

We'll use a joining table called feed_follows to accomplish this. Creating a feed_follow record indicates that a user is now following a feed. Deleting it is the same as "unfollowing" a feed.

Assignment
Create a feed_follows table with a new migration. It should:
Have an id column that is a primary key.
Have created_at and updated_at columns.
Have user_id and feed_id foreign key columns. Feed follows should auto delete when a user or feed is deleted.
Add a unique constraint on user/feed pairs - we don't want duplicate follow records.
Add a CreateFeedFollow query. It will be a deceptively complex SQL query. It should insert a feed follow record, but then return all the fields from the feed follow as well as the names of the linked user and feed. I'll add a tip at the bottom of this lesson if you need it.
Add a follow command. It takes a single url argument and creates a new feed follow record for the current user. It should print the name of the feed and the current user once the record is created (which the query we just made should support). You'll need a query to look up feeds by URL.
Add a GetFeedFollowsForUser query. It should return all the feed follows for a given user, and include the names of the feeds and user in the result.
Add a following command. It should print all the names of the feeds the current user is following.
Enhance the addfeed command. It should now automatically create a feed follow record for the current user when they add a feed.

# 4.2 Middleware

We have 3 command handlers (and we'll add more) that all start by ensuring that a user is logged in.

addfeed
follow
following
They all share this code (or something similar):

user, err := s.db.GetUser(context.Background(), s.cfg.CurrentUserName)
if err != nil {
 return err
}

Let's create some "middleware" that abstracts this away for us. In addition, if we need to modify this code for any reason later, there will be only one place that must be edited.

Middleware is a way to wrap a function with additional functionality. It is a common pattern that allows us to write DRY code.
Assignment
Create logged-in middleware. It will allow us to change the function signature of our handlers that require a logged in user to accept a user as an argument and DRY up our code. Here's the function signature of my middleware:

middlewareLoggedIn(handler func(s *state, cmd command, user database.User) error) func(*state, command) error

You'll notice it's a higher order function that takes a handler of the "logged in" type and returns a "normal" handler that we can register. I used it like this:

cmds.register("addfeed", middlewareLoggedIn(handlerAddFeed))

Test your code before and after this refactor to make sure that everything still works.

Run and submit the CLI tests.

# 4.3 Unfollow

If you can follow feeds, it only makes sense to be able to unfollow them.

Assignment
Add a new SQL query to delete a feed follow record by user and feed id combination.
Add a new unfollow command that accepts a feed's URL as an argument and unfollows it for the current user. This is, of course, a "logged in" command - use the new middleware.

# 5.1 Aggregate

Feeds are essentially just lists of posts. A post represents a single web page. The entire point of the gator program is to fetch the actual posts from the feed URLs and store them in our database. That way we can display them nicely in our CLI.

Assignment
Enhance the agg command to actually fetch the RSS feeds, parse them, and print the posts to the console--all in a long-running loop.

Create a new migration that adds a last_fetched_at column to the feeds table. It should be nullable.
Add a MarkFeedFetched SQL query. It should simply set the last_fetched_at and updated_at columns to the current time for a given feed (probably by ID is simplest).
Add a GetNextFeedToFetch SQL query. It should return the next feed we should fetch posts from. We want to scrape all the feeds in a continuous loop. A simple approach is to keep track of when a feed was last fetched, and always fetch the oldest one first (or any that haven't ever been fetched). SQL has a NULLS FIRST clause that can help with this.
Write an aggregation function, I called mine scrapeFeeds. It should:
Get the next feed to fetch from the DB.
Mark it as fetched.
Fetch the feed using the URL (we already wrote this function)
Iterate over the items in the feed and print their titles to the console.
Update the agg command to now take a single argument: time_between_reqs.
time_between_reqs is a duration string, like 1s, 1m, 1h, etc. I used the time.ParseDuration function to parse it into a time.Duration value.

It should print a message like Collecting feeds every 1m0s when it starts.
Use a time.Ticker to run your scrapeFeeds function once every time_between_reqs. I used a for loop to ensure that it runs immediately (I don't like waiting) and then every time the ticker ticks:
ticker := time.NewTicker(timeBetweenRequests)
for ; ; <-ticker.C {
 scrapeFeeds(s)
}

Do NOT DOS the servers you're fetching feeds from. Anytime you write code that makes a request to a third party server you should be sure that you are not making too many requests too quickly. That's why I recommend printing to the console for each request, and being ready with a quick Ctrl+C to stop the program if you see something going wrong.
The agg command should now be a never-ending loop that fetches feeds and prints posts to the console. The intended use case is to leave the agg command running in the background while you interact with the program in another terminal.

You should be able to kill the program with Ctrl+C.

There are no CLI tests for this lesson, test your own program and make sure everything behaves as expected. Here are a few RSS feeds to get you started:

TechCrunch: <https://techcrunch.com/feed/>
Hacker News: <https://news.ycombinator.com/rss>
Boot.dev Blog: <https://blog.boot.dev/index.xml>

**important**

psql "postgres://carlosinfante:@localhost:5432/gator"

goose postgres "postgres://carlosinfante:@localhost:5432/gator" up

goose -dir sql/schema postgres "postgres://carlosinfante:@localhost:5432/gator" status


