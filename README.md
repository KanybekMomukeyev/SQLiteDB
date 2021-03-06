# SQLiteDB

SQLiteDB is a simple and lightweight SQLite wrapper for Swift. It allows all basic SQLite functionality including being able to bind values to parameters in an SQL statement. You can either include an SQLite database file with your project (in case you want to pre-load data) and have it be copied over automatically in to your documents folder, or have the necessary database and the relevant table structures created automatically for you via SQLiteDB.

**Update: (11 Apr 2017)** The latest version of SQLiteDB has the option to specify the database name (it defaults to "data.db") and so you must open the database explicitly via a call to `openDB()` before you run any queries or execute commands. Please be aware of this change when updating an existing project. (See the included iOS sample project's `AppDelegate.swift` file for an example.)

**Important:** If you are new to Swift or have not bothered to read up on the Swift documentation, please do not ask me to explain Swift functionality to you. Instead, please do some reading into Swift functionality since there is plenty of excellent documentaiton on the subject. On the other hand, if you're not looking for free advice but are willing to pay for my time, do feel free to contact me :)

## Adding to Your Project

* If you want to pre-load data or have the table structures and indexes pre-created, or, if you are not using `SQLTable` sub-classes but are instead using `SQLiteDB` directly, then you need to create an SQLite database file to be included in your project.

  Create your SQLite database however you like, but name it `data.db` and then add the `data.db` file to your Xcode project. (If you want to name the database file something other than `data.db`, then set the `DB_NAME` property in the `SQLiteDB` class accordingly.)

    **Note:** Remember to add the database file above to your application target when you add it to the project. If you don't add the database file to a project target, it will not be copied to the device along with the other project resources.
    
  If you do not want to pre-load data and are using `SQLTable` sub-classes to access your tables, you can skip the above step since SQLiteDB will automatically create your table structures for you if the database file is not included in the project. However, in order for this to work, you need to pass `false` as the parameter for the `openDB` method when you inovke it, like this:
	
  	```swift
  	db.openDB(copyFile:false)
  	```
	
* Add all of the included source files (except for README.md, of course) to your project.

* If you don't have a bridging header file, use the included `Bridging-Header.h` file. If you already have a bridging header file, then copy the contents from the included `Bridging-Header.h` file to your own bridging header file.

* If you didn't have a bridging header file, make sure that you modify your project settings to point to the new bridging header file. This will be under  **Build Settings** for your target and will be named **Objective-C Bridging Header**.

* Add the SQLite library (libsqlite3.0.dylib or libsqlite3.tbd, depending on your Xcode version) to your project under **Build Phases** - **Link Binary With Libraries** section.

That's it. You're set!

## Usage

There are two ways you can use `SQLiteDB` in your project:

### Basic

You can use the `SQLiteDB` class directly to get a reference to the database and then run queries (or execute statements) on the database directly.

* You can gain access to the shared database instance as follows:

  	```swift
  	let db = SQLiteDB.shared
  	```

* Before you make any SQL queries, or execute commands, you should open the SQLite database. In most cases, this needs to be done only once per application and so you can do it in your `AppDelegate`, for example:
	
  	```swift
  	db.openDB()
  	```
 
* You can make SQL queries using the `query` method (the results are returned as an array of dictionaries where the key is a `String` and the value is of type `Any`):

  	```swift
  	let data = db.query(sql:"SELECT * FROM customers WHERE name='John'")
  	let row = data[0]
  	if let name = row["name"] {
  		textLabel.text = name as! String
  	}
  	```
In the above, `db` is a reference to the shared SQLite database instance. You can access a column from your query results by subscripting a row of the returned results (the rows are dictionaries) based on the column name. That returns an optional `Any` value which you can cast to the relevant data type.

* If you'd prefer to bind values to your query instead of creating the full SQL statement, then you can execute the above SQL also like this:

  	```swift
  	let name = "John"
  	let data = db.query(sql:"SELECT * FROM customers WHERE name=?", parameters:[name])
  	```

* Of course, you can also construct the above SQL query by using Swift's string manipulation functionality as well (without using the SQLite bind functionality):

  	```swift
  	let name = "John"
  	let data = db.query(sql:"SELECT * FROM customers WHERE name='\(name)'")
  	```

* You can execute all non-query SQL commands (INSERT, DELETE, UPDATE etc.) using the `execute` method:

  	```swift
  	let result = db.execute(sql:"DELETE FROM customers WHERE last_name='Smith'")
  	// If the result is 0 then the operation failed, for inserts the result gives the newly inserted record ID
  	```

* The `esc` method which was previously available in SQLiteDB is no longer there. So, for instance, if you need to escape strings with embedded quotes, you should use the SQLite parameter binding functionality as shown above.

### Using `SQLTable`

If you would prefer to model your database tables as classes and do any data access via the classes, SQLiteDB also provides an `SQLTable` class which does most of the heavy lifting for you. 

If you create a sub-class of `SQLTable`, define properties where the names match the column names in your SQLite table, then you can use the sub-class to save to/update the database without having to write all the necessary boilerplate code yourself.

Additionally, with this approach, you don't need to include an SQLite database project with your app (unless you want to). SQLiteDB will infer the structure for the tables based on your `SQLTable` sub-classes and automatically create the necessary tables for you, if they aren't present.

For example, say that you have a `Categories` table with just two columns - `id` and `name`. Then, the `SQLTable` sub-class definition for the table would look something like this:

```swift
class Category:SQLTable {
	var id = -1
	var name = ""
}
```

It's as simple as that! You don't have to write any insert, update, or delete methods since `SQLTable` handles all of that for you behind the scenese :)

**Note:** Do note that for a table named `Categories`, the class has to be named `Category` - the table name has to be plural, and the class name has to be singular.

The only additional thing you need to do when you use `SQLTable` sub-classes and want the table structures to be automatically created for you is that you have to specify that you don't want to create a copy of a database in your project resources when you invoke `openDB`. So you have to have you `openDB` command be something like this:

```swift
db.openDB(copyFile:false)
```
	
Once you do that, you can run any SQL queries or execute commands on the database without any issues. 

Here are some quick examples of how you use the `Category` class from the above example:

* Add a new `Category` item to the table:

```swift
let category = Category()
category.name = "My New Category"
_ = category.save()
```

The save method returns a non-zero value if the save was successful. In the case of a new record, the return value is the `id` of the newly inserted row. You can check the return value to see if the save was sucessful or not since a 0 value means that the save failed for some reason.

* Get a `Category` by `id`:

```swift
if let category = Category.rowBy(id:10) as? Category {
	NSLog("Found category with ID = 10")
}
```

* Query the `Category` table:

```swift
let array = Category.rows(filter:"id > 10") as! [Category]
```

* Get a specific `Category` row (to display categories via a `UITableView`, for example):

```swift
if let category = row(number:1) as? Category {
	NSLog("Got first un-ordered category row")
}
```

* Delete a `Category`:

```swift
if let category = Category.rowBy(id:10) as? Category {
	category.delete()
	NSLog("Deleted category with ID = 10")
}
```

You can refer to the sample iOS and macOS projects for more examples of how to implement data access using `SQLTable`.

## Questions?

* FAQ: [FAQs](https://github.com/FahimF/SQLiteDB/wiki/FAQs)
* [IMG](https://www.codementor.io/fahimfarook?utm_source=github&utm_medium=button&utm_term=fahimfarook&utm_campaign=github)
* Web: [http://rooksoft.sg/](http://rooksoft.sg/)
* Twitter: [http://twitter.com/FahimFarook](http://twitter.com/FahimFarook)

SQLiteDB is under DWYWPL - Do What You Will Public License :) Do whatever you want either personally or commercially with the code but if you'd like, feel free to attribute in your app.