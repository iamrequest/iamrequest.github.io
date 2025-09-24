# About
- What is a database, why would you need one
- What is the asset
- What is SQLite

This article is mainly meant to get you off the ground with using databases in Godot. If you'd rather watch a video about this topic, I recommend [FinePointCGI's video on this plugin](https://www.youtube.com/watch?v=j-BRiTrw_F0).

## What is a Database?

A database is one way of storing data, in the same way that you might use a flat file (eg: CSV file, JSON file, etc). 

The benefit to using a database instead of a simple text file is in efficient read/writes, data organization, and super quick queries (eg: select all high score entries with a high score above 100, and sort them alphabetically).

## About SQLite

[SQLite](https://www.sqlite.org/) is an open source database library. From the SQLite home page: 

> SQLite is a C-language library that implements a small, fast, self-contained, high-reliability, full-featured, SQL database engine. SQLite is the most used database engine in the world. SQLite is built into all mobile phones and most computers and comes bundled inside countless other applications that people use every day.

Unless you have a use-case that requires a feature from another type of database, I recommend this one. Remember to [keep it simple, stupid!](https://en.wikipedia.org/wiki/KISS_principle)

# Setup

These install instructions are valid as of September 23, 2025. You're probably good to follow these instructions, but it's worth checking the github page to be safe.

- In the Godot Asset Store, search for `Godot-SQLite`, and download it.
- Open the Project Settings window (`Project -> Project Settings`). Click on the `Plugins` tab, and enable the `Godot SQLite` plugin.

> installation 1 img
	Download the plugin from the asset store
> installation 2 img
	Enable the plugin in the Project Settings window.

You may want to also install a database browser, so you can inspect your data. I recommend [DB Browser for SQLite (DB4S)](https://sqlitebrowser.org/).

# Creating a Database

Your SQLite database will be stored in a `.db` file. You can save this anywhere, but for now we can write it to the `user://` directory.

```python
	var db: SQLite
	var _db_path := "user://player_data.db"

	func open_database() -> bool:
		db = SQLite.new()
		db.path = _db_path

		var was_db_opened_successfully = db.open_db()

		# Handle error case as you see fit
		return was_db_opened_successfully
```

This will open the database file, and will also create the file if it doesn't already exist.

## About user:// vs res://

You likely want to store your database file as a child of the `user://` directory, rather than `res://`. From the [Godot documentation](https://docs.godotengine.org/en/stable/tutorials/io/data_paths.html):

| To store persistent data files, like the player's save or settings, you want to use user:// instead of res:// as your path's prefix. This is because when the game is running, the project's file system will likely be read-only.
| 
| The user:// prefix points to a different directory on the user's device. Unlike res://, the directory pointed at by user:// is created automatically and guaranteed to be writable to, even in an exported project.


# Creating Tables

You can create a table either in a database browser, or via code.

First, we define the columns in a dictionary.
```python
var column_definitions = {
	# Primary Key
	"id": {
		"data_type": "int",
		"primary_key": true,
		"not_null": true,
		"auto-increment": true
	},

	# Foreign Key (see note below)
	"album_id": { 
		"data_type": "int" ,

		# Assume that the table "albums" exists, with a primary key of "album_id"
		"foreign_key": "albums.album_id" ,
	},

	# Some example fields
	"song_name": { 
		"data_type": "text",

		# Defines a UNIQUE constraint on the column
		"unique": false,

		# Defines the default value of the column, if none is supplied
		"default": "DefaultSong"
	},

	# For byte arrays (images, raw data, etc)
	"album_art": { "data_type": "blob" },

	# For floats/doubles
	"bpm": { "data_type": "real" },
}


```

Then, we create the table in the database:

```python
var my_table_name := "songs"
db.create_table(my_table_name, column_definitions)
```


Note: To make the database enforce the relationship of foreign keys, you have to manually set `db.foreign_keys = true`. This is recommended, to ensure data integrity in your database.

As of September 2025, these are the data types available to you when creating tables. Any data types not found in the table will throw an exception if you try to use them. You can find the full, up to date list of data types on the [github page](https://github.com/2shady4u/godot-sqlite).

|Data Type|SQLite|Godot|
|---------|------|-----|
|int|INTEGER|TYPE\_INT|
|real|REAL|TYPE\_REAL|
|text|TEXT|TYPE\_STRING|
|char(?)\*|CHAR(?)|TYPE\_STRING|
|blob|BLOB|TYPE\_PACKED\_BYTE\_ARRAY|

\*: The question mark should be replaced with the maximum number of characters usable in the column.


# Basic operations (CRUD)

The 4 types of database queries you'll typically be doing are CRUD operations: Create, Read, Update, Delete.

## Create (Insert)

```
var row_data = {
	"song_name": "Perpetual Terminal",
	"bpm": 128
}

db.insert_row("songs", row_data)
```

## Read Data (Select)

Below is a simple example of doing a SELECT statement on a single table. A more complex example that includes JOIN statements and parameter binding can be found further below.

```python
var table_name := "songs"
var where_clause := "bpm > 120"
var columns := [ "song_name", "bpm" ]
var results = db.select_rows(table_name, where_clause, columns)

print(results)
[
	{ "name": "Perpetual Terminal", "bpm": 128 },
	{ "name": "Marigold", "bpm": 213 },
]
```

You might iterate over the results like so:

```python
for row in results:
	print("%s: %s" % [ row["name"], row["bpm"] ])
```


## Update Data

```python
var table_name := "songs"
var where_clause := "id = 1"
var row_data := [ 
	"song_name", "Free Rein to Passions" 
	"bpm", "135" 
]

var was_successful: bool = db.update_rows(table_name, where_clause, row_data)
```

## Delete Data

```python
var table_name := "songs"
var where_clause := "id = 1"

var was_successful: bool = db.delete_rows(table_name, where_clause)
```

# Advanced Queries

## Joins

You will often need to fetch and filter data from more than one table. This is where Joins will come in.

```python
# 1. Build your query
var query_string := "
	SELECT songs.name as song_name, 
		songs.bpm as bpm, 
		albums.name as album_name,
		albums.release_date
	FROM
		songs
			LEFT JOIN albums ON songs.id = albums.id
	WHERE
		albums.name = 'Darkest Hour'"

# 2. Perform the query
var was_successful: bool = db.query(query_string)

# 3. Fetch the query results via the array db.query_result()
for row in db.query_result:
	print(row)
	# ...
```

Example result:

```python
[
	{ 
		"song_name": "Unfamiliar Sun",
		"album_name": "Lookout Low",
		"release_date": 2019
	},
	{ 
		"song_name": "DVP",
		"album_name": "The Dream Is Over",
		"release_date": 2016
	}
]
```


Warning! This above approach to making queries is vulnerable to  and is not recommended if you're planning on using a query that includes user input. The issue is that if we don't sanitize the input going into our queries, users will be able to execute arbitrary SQL code (bad!). 

## Sanitizing Query Input (Preventing SQL Injections)

I recommend using `db.query_with_bindings()` instead of `db.query()` anywhere that you plan on including user input. It works almost exactly the same, with a small change:

```python
# 1. Create the query string in the same way, substituting user input for a question mark
var query_string := "
	SELECT 
		# [... ]
	WHERE
		albums.name = ?;" 


# Instead of putting user input directly in the query string, we instead add it to an array
var album_name = _album_name_text_field.text
var query_bindings = [ album_name ] 


# 2. Perform the query
var was_successful: bool = db.query_with_bindings(query_string, query_bindings)

# 3. Fetch the query results in the same way as you would with db.query_result()
for row in db.query_result:
	print(row)
	# ...
```


# Credits

- [SQLite](https://www.sqlite.org/)
- [Godot SQLite](https://github.com/2shady4u/godot-sqlite)
- [FinePointCGI: Integrating SQLite into Godot 4.2](https://www.youtube.com/watch?v=j-BRiTrw_F0)
