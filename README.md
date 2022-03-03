# en traduccion
# Using Sequelize ORM with Node.js and Express 
# El ORM Sequelize ORM con Node.js & Express 

Receta de [**here**](https://stackabuse.com/using-sequelize-orm-with-nodejs-and-express/)
Codigo original en [**GitHub**](https://github.com/StackAbuse/sequelize-express-notes-app)

### Intro

[Sequelize](https://github.com/sequelize/sequelize/) es un ORM popular sobre Node.js, y en este tutorial lo aplicamos a un API CRUD API para manejar notas.

@@@@@

Interacting with databases is a common task for backend applications. This was typically done via raw SQL queries, which can be difficult to construct, especially for those new to SQL or databases in general.

Eventually, *Object Relational Mappers* (ORMs) came to be - designed to make managing databases easier. They automatically map out the objects (entities) from our code in a relational database, as the name implies.

No longer would we write raw SQL queries and execute them against the database. By providing us with a programmatic way to connect our code to the database and manipulate the persisted data, we can focus more on the business logic and less on error-prone SQL.

### What is an ORM?

*Object Relational Mapping* is a technique that maps software objects to database tables. Developers can interact with objects instead of having to actually write any database queries. When an object is read, created, updated, or deleted, the ORM constructs and executes a database query under the hood.

Another advantage of ORMs is they support multiple databases: [Postgres](https://www.postgresql.org/), [MySQL](https://www.mysql.com/), [SQLite](https://www.sqlite.org/index.html), etc. If you write an application using raw queries, it will be difficult to move to a different database because many of the queries will need to be re-written.

With an ORM, switching databases is done by the ORM itself, and typically all you need to do is change a value or two in a configuration file.

### Sequelize

There are many Node ORMs, including the popular [Bookshelf.js](/bookshelf-js-a-node-js-orm/) and [TypeORM](https://stackshare.io/typeorm).

> So, why and when to choose Sequelize?

+ Firstly, it has been around for a long time - 2011. It has thousands of GitHub stars and is used by tons of applications. Due to it's age and popularity it is stable and has plenty of documentation available online.

+ In addition to it's maturity and stability, Sequelize has a large feature set that covers: queries, scopes, relations, transactions, raw queries, migrations, read replication, etc. 

+ Sequelize is promise-based, making it ***easier to manage asynchronous functions and exceptions***. It also supports all the popular SQL dialects: PostgreSQL, MySQL, MariaDB, SQLite, and MSSQL.

On the other hand, there's no NoSQL support which can be seen in ORMs (or Object *Document* Mappers, in this case) such as [Mongoose](https://mongoosejs.com/). Really, deciding which ORM to choose depends mainly on the requirements of the project you're working on.

### Installing Sequelize {#installingsequelize}

**Note**: If you want to follow along with the code, you can find it [here on GitHub](https://github.com/StackAbuse/sequelize-express-notes-app).

Let's make a skeleton Node application and install Sequelize. First off, let's create a directory for our project, enter it, and create a project with the default settings:

```bash
mkdir notes-app
cd notes-app
npm init -y
```

Next we'll create the application file with a basic Express server and router. Let's call it `index.js` to matches the default filename from `npm init`:

Next, to easily create a web server we'll install Express:

```bash
npm install --save express
```

And with it installed, let's set up the server:

```javascript
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => res.send('Notes App'));

app.listen(port, () => console.log(`notes-app listening on port ${port}!`));
```

Finally, we can go ahead and install Sequelize and our database of choice via `npm`:

```bash
$ npm install --save sequelize
$ npm install --save sqlite3
```

It doesn't matter which database you use as Sequelize is database-agnostic. The way we use it is the same, no matter the underlying database. SQLite3 is easy to work with for local development and is a popular choice for those purposes.

Now, let's add some code to the `index.js` file to set up the database and check the connection using Sequelize. Depending on which database you're using, you may need to define a different dialect:

```javascript
const Sequelize = require('sequelize');
const sequelize = new Sequelize({
  // The `host` parameter is required for other databases
  // host: 'localhost'
  dialect: 'sqlite',
  storage: './database.sqlite'
});
```

After importing Sequelize, we set it up with the parameters it requires to run. You can also add more parameters here, such as the `pool`, though what we have is enough to get started. The `dialect` depends on which database you're using, and the `storage` simply points to the database file.

The `database.sqlite` file is created automatically at the root level of our project.

**Note:** It's worth checking the [Sequelize Docs](https://sequelize.org/v5/manual/getting-started.html) for setting up different databases and the required information for each.

If you're using MySQL, Postgres, MariaDB, or MSSQL, instead of passing each parameter separately, you can also just pass the connection URI:

```javascript
const sequelize = new Sequelize('postgres://user:[email protected]:5432/dbname');
```

Finally, let's test the connection by running the `.authenticate()` method. Under the hood, it simply runs a `SELECT` query and checks if the database responds correctly:

```javascript
sequelize
  .authenticate()
  .then(() => {
    console.log('Connection has been established successfully.');
  })
  .catch(err => {
    console.error('Unable to connect to the database:', err);
  });
```

Running the application, we're greeted with:

```bash
node index.js
notes-app listening on port 3000!
Executing (default): SELECT 1+1 AS result
Connection has been established successfully.
```

### Creating a Model for Mapping

Before we can build a notes API we need to create a notes table. To do that we need to define a `Note` model, which we'll assign to a constant so that it can be used throughout our API. In the `define` function we specify the table name and fields. In this case a text field for the note and a string for tag:

As with relational databases, before building an API, we'll need to create adequate tables first. Since we want to avoid creating it by hand
using SQL, we'll define a `Model` class and then have Sequelize map it out into a table.

This can be done by either extending the `Sequelize.Model` class and running the `.init()` function, passing parameters, or by defining a `const` and assigning it the returned value of the `.define()` method from Sequelize.

The latter is more concise, so we'll be going with that one: 

```javascript
const Note = sequelize.define('notes', { note: Sequelize.TEXT, tag: Sequelize.STRING });
```

#### Mapping the Model to the Database

Now that we have a `Note` model we can create the `notes` table in the database. In a production application we'd normally make database changes via
[migrations](https://sequelize.org/v5/manual/migrations.html) so that changes are tracked in source control.

Though, to keep things concise, we'll use the `.sync()` method. What the `.sync()` does is simple - it synchronizes all the defined models to the database:

```javascript
sequelize.sync({ force: true })
  .then(() => {
    console.log(`Database & tables created!`);
  });
```

Here, we've used the `force` flag and set it to `true`. If a table exists already exists, the method will `DROP` it and `CREATE` a new one. If it doesn't exist, a table is just created.

Finally, let's create some sample notes that we'll then persist in the database:

```javascript
sequelize.sync({ force: true })
  .then(() => {
    console.log(`Database & tables created!`);

    Note.bulkCreate([
      { note: 'pick up some bread after work', tag: 'shopping' },
      { note: 'remember to write up meeting notes', tag: 'work' },
      { note: 'learn how to use node orm', tag: 'work' }
    ]).then(function() {
      return Note.findAll();
    }).then(function(notes) {
      console.log(notes);
    });
  });
```

Running the server, our notes are printed out in the console, as well as the SQL operations performed by Sequelize. Let's connect to the database to verify that the records have indeed been added properly:

```bash
$ sqlite3 database.sqlite
sqlite> select * from notes;
1|pick up some bread after work|shopping|2020-02-21 18:24:19.402 +00:00|2020-02-21 18:24:19.402 +00:00
2|remember to write up meeting notes|work|2020-02-21 18:24:19.402 +00:00|2020-02-21 18:24:19.402 +00:00
3|learn how to use node orm|work|2020-02-21 18:24:19.402 +00:00|2020-02-21 18:24:19.402 +00:00
sqlite> .exit
$
```

With the database in place and our table(s) created, let's go ahead and implement the basic CRUD functionality.

### Reading Entities

Our model, `Note`, now has built-in methods that help us perform operations on the persisted records in the database.

#### Read All Entities

For example, we can read all records of that class saved by using the `.findAll()` method. Let's make a simple endpoint that serves all the persisted entities:

```javascript
app.get('/notes', function(req, res) {
  Note.findAll().then(notes => res.json(notes));
});
```

The `.findAll()` method returns an array of notes, which we can use to render a response body, via `res.json`.

Let's test the endpoint via `curl`:

```bash
$ curl http://localhost:3000/notes
[{"id":1,"note":"pick up some bread after work","tag":"shopping","createdAt":"2020-02-27T17:02:10.881Z","updatedAt":"2020-02-27T17:02:10.881Z"},{"id":2,"note":"remember to write up meeting notes","tag":"work","createdAt":"2020-02-27T17:02:10.881Z","updatedAt":"2020-02-27T17:02:10.881Z"},{"id":3,"note":"learn how to use node orm","tag":"work","createdAt":"2020-02-27T17:02:10.881Z","updatedAt":"2020-02-27T17:02:10.881Z"}]
$
```

As you can see, all of our database entries were returned to us, but in JSON form.

Though, if we're looking to add a bit more functionality, we've got query operations such as `SELECT`, `WHERE`, `AND`, `OR`, and `LIMIT` supported by this method.

A full list of supported query methods can be found on the [Sequelize Docs](https://sequelize.org/v5/manual/querying.html) page.

#### Read Entities WHERE

With that in mind, let's make an endpoint that serves a single, specific note:

```javascript
app.get('/notes/:id', function(req, res) {
  Note.findAll({ where: { id: req.params.id } }).then(notes => res.json(notes));
});
```

The endpoints accepts an `id` parameter, used to look up a note via the `WHERE` clause. Let's test it out via `curl`:

```sh
$ curl http://localhost:3000/notes/2
[{"id":2,"note":"remember to write up meeting notes","tag":"work","createdAt":"2020-02-27T17:03:17.592Z","updatedAt":"2020-02-27T17:03:17.592Z"}]
$
```

**Note**: Since this route uses a wildcard param, `:id`, it will match *any* string that comes after `/notes/`. For this reason, this route should be at the *end* of your index.js file. This allows other routes, like `/notes/search`, to handle a request before `/notes/:id` picks it up. Otherwise the `search` keyword in the URL path will be treated like an ID.

#### Read Entities WHERE AND

For even more specific queries, let's make an endpoint utilizing both `WHERE` and `AND` statements:

```javascript
app.get('/notes/search', function(req, res) {
  Note.findAll({ where: { note: req.query.note, tag: req.query.tag } }).then(notes => res.json(notes));
});
```

Here, we're looking for notes that match both the `note` and `tag` specified by the parameters. Again, let's test it out via `curl`:

```sh
$ curl "http://localhost:3000/notes/search?note=pick%20up%20some%20bread%20after%20work&tag=shopping"
[{"id":1,"note":"pick up some bread after work","tag":"shopping","createdAt":"2020-02-27T17:09:53.964Z","updatedAt":"2020-02-27T17:09:53.964Z"}]
$
```

#### Read Entities OR

If we're trying to be a bit more vague, we can use the `OR` statement and search for notes that match *any* of the given parameters. Change the `/notes/search` route to:

```javascript
const Op = Sequelize.Op;

app.get('/notes/search', function(req, res) {
  Note.findAll({
    where: {
      tag: {
        [Op.or]: [].concat(req.query.tag)
      }
    }
  }).then(notes => res.json(notes));
});
```

Here we're using `Sequelize.Op` to implement an `OR` query. Sequelize provides several operators to choose from such as `Op.or`, `Op.and`, `Op.eq`, `Op.ne`, `Op.is`, `Op.not`, etc. These are mainly used to create more complex operations, like querying with a regex string.

Note that we're using `req.query.tag` as the argument to `.findAll()`. Sequelize expects an array here, so we force `tag` to be an array using `[].concat()`. In our test below we'll pass multiple arguments in our request URL:

```sh
$ curl "http://localhost:3000/notes/search?tag=shopping&tag=work"
[{"id":1,"note":"pick up some bread after work","tag":"shopping","createdAt":"2020-02-27T17:11:27.518Z","updatedAt":"2020-02-27T17:11:27.518Z"},{"id":2,"note":"remember to write up meeting notes","tag":"work","createdAt":"2020-02-27T17:11:27.518Z","updatedAt":"2020-02-27T17:11:27.518Z"},{"id":3,"note":"learn how to use node orm","tag":"work","createdAt":"2020-02-27T17:11:27.518Z","updatedAt":"2020-02-27T17:11:27.518Z"}]
$
```

When passing the same query param multiple times like this, it will show up as an array in the `req.query` object. So in the above example, `req.query.tag` is `['shopping', 'work']`.

#### Read Entities LIMIT

The last thing we'll cover in this section is `LIMIT`. Let's say that we wanted to modify the pervious query to only return two results max. We'll do this by adding the `limit` parameter and assigning it a positive integer:

```javascript
const Op = Sequelize.Op;

app.get('/notes/search', function(req, res) {
  Note.findAll({
    limit: 2,
    where: {
      tag: {
        [Op.or]: [].concat(req.query.tag)
      }
    }
  }).then(notes => res.json(notes));
});
```

You can see a full list of query functions at the [Sequelize docs](https://sequelize.org/v5/manual/querying.html).

### Inserting Entities

Inserting entities is a lot more straightforward as there's really no two ways to perform this operation.

Let's add a new endpoint for adding notes:

```javascript
const bodyParser = require('body-parser');
app.use(bodyParser.json());

app.post('/notes', function(req, res) {
  Note.create({ note: req.body.note, tag: req.body.tag }).then(function(note) {
    res.json(note);
  });
});
```

The `body-parser` module is required for the endpoint to accept and parse JSON parameters. You don't need to explicitly install the `body-parser` package because it's already included with Express.

Inside the route we're using the `.create()` method to insert a note into the database, based on the passed parameters.

We can test it out with another `curl` request:

```sh
$ curl -d '{"note":"go the gym","tag":"health"}' -H "Content-Type: application/json" -X POST http://localhost:3000/notes
{"id":4,"note":"go the gym","tag":"health","updatedAt":"2020-02-27T17:13:42.281Z","createdAt":"2020-02-27T17:13:42.281Z"}
$
```

Running this request will result in the creation of a note in our database and returns the new database object to us.

### Updating Entities 

Sometimes, we'd wish to update already existing entities. To do this, we'll rely on the `.update()` method on the result of the `.findByPk()` method:

```javascript
app.put('/notes/:id', function(req, res) {
  Note.findByPk(req.params.id).then(function(note) {
    note.update({
      note: req.body.note,
      tag: req.body.tag
    }).then((note) => {
      res.json(note);
    });
  });
});
```
The `.findByPk()` method is also an inherited method in our model class. It searches for an entity with the given primary key. Essentially, it's easier to return single entities by their ID using this method than writing a `SELECT WHERE` query.

Given the returned entity, we run the `.update()` method to actually put the new values in place. Let's verify this via `curl`:

```sh
$ curl -X PUT -H "Content-Type: application/json" -d '{"note":"pick up some milk after work","tag":"shopping"}' http://localhost:3000/notes/1
{"id":1,"note":"pick up some milk after work","tag":"shopping","createdAt":"2020-02-27T17:14:55.621Z","updatedAt":"2020-02-27T17:14:58.230Z"}
$
```

Firing this request updates the first note with new content and returns the updated object:

#### Deleting Entities

And finally, when we'd like to delete records from our database, we use the `.destroy()` method on the result of the `.findByPk()` method:

```javascript
app.delete('/notes/:id', function(req, res) {
  Note.findByPk(req.params.id).then(function(note) {
    note.destroy();
  }).then((note) => {
    res.sendStatus(200);
  });
});
```

The route for `.delete()` looks similar to `.update()`. We use `.findByPk()` to find a specific note by ID. Then, the `.destroy()` method removes the note from the database.

Finally, a `200 OK` response is returned to the client.

### Conclusion

*Object Relational Mapping* (ORM) is a technique that maps software objects to database tables. Sequelize is a popular and stable ORM tool used alongside Node.js. In this article, we've discussed what ORMs are, how they work and what are some advantages of using them over writing raw queries.

With that knowledge, we proceeded to write a simple Node.js/Express app that uses Sequelize to persist a `Note` model to the database. Using the inherited methods, we've then performed CRUD operations on the database.


