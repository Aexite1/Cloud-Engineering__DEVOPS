
# Building and Deploying a MEAN Book Register

The MEAN stack combines four JavaScript technologies into one full-stack development workflow. In this project, the stack is used to build a simple Book Register application that stores book information, exposes it through an Express API, and provides a browser interface with AngularJS.

### Application flow

The implementation follows this path:

`Browser (AngularJS) → Express/Node.js → Mongoose → MongoDB`

The EC2 instance hosts the application, while MongoDB provides persistent storage for the book records.


**MongoDB:** The database layer. Book records are stored as documents, giving the application a flexible NoSQL data model.

**Express.js:** The server framework used to define HTTP routes, process requests, and connect the browser-facing application to the database layer.

**AngularJS:** The client-side framework used to make the Book Register page interactive. It handles the interface state and communicates with the Express endpoints.

**Node.js:** The runtime on which the backend executes, allowing JavaScript to be used for the server as well as the client.


## 1. Provision the AWS Server

An EC2 instance was created from the AWS console using Ubuntu 22.04 LTS in the `us-north-1b` region. The selected instance size was `t3.small`.

![Launch Instance](<./images/Screenshot 2026-08-11 181332.png>)
![Launch Instance](<./images/Screenshot 2026-08-11 181922.png>)

The instance was associated with the `ec2-key` SSH key so that it could be administered remotely through SSH on port 22.

The EC2 security group was configured to expose the ports required by the application:

- **80/TCP:** HTTP traffic from the internet.

- **443/TCP:** HTTPS traffic from the internet.

- **22/TCP:** SSH access for administration.

- **3300/TCP:** The port used by the Book Register server.

![Security Rules](<./images/Screenshot 2026-08-11 182416.png>)

Before connecting, the private key permissions were restricted and the key was then used to establish the SSH session:
```bash
chmod 400 ec2-key.pem
```
```bash
ssh -i "my-ec2-key.pem" ubuntu@51.20.64.24
```
In the example above, the SSH account is `ubuntu` and the public address is `51.20.64.24`. A real deployment should use the address assigned to its own EC2 instance.

![Connect to instance](<./images/Screenshot 2026-08-11 183023.png>)


## 2. Prepare the Node.js Runtime

Node.js provides the runtime required for the Express backend and the JavaScript tooling used by the project.

### Update the Ubuntu system

```bash
sudo apt update && sudo apt upgrade -y
```
![Update ubuntu](<./images/Screenshot 2026-08-11 183405.png>)

### Install the supporting packages

```bash
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates

curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -  ```

![Add cert.](<./images/Screenshot 2026-08-11 190810.png>)
![Add cert.](<./images/Screenshot 2026-08-11 191213.png>)
### Install Node.js

```bash
sudo apt-get install -y nodejs
```
![Install nodejs](<./images/Screenshot 2026-08-11 191403.png>)


## 3. Install and Start MongoDB

The database stores four pieces of information for each book: its name, ISBN, author, and number of pages.

### Add MongoDB's signing key

```bash
sudo apt install gnupg && curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-archive-keyring.gpg
```
![MongoDB's signing key](<./images/Screenshot 2026-08-12 010730.png>)

### Register the MongoDB package repository

```bash
echo "deb [ signed-by=/usr/share/keyrings/mongodb-archive-keyring.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```
![Mongo public key](<./images/Screenshot 2026-08-12 010946.png>)


### Install the MongoDB server packages

```bash
sudo apt-get update
```
![Update server](<./images/Screenshot 2026-08-12 011708.png>)

```bash
sudo apt-get install -y mongodb-org
```
![Install mongodb](<./images/Screenshot 2026-08-12 011809.png>)


### Start MongoDB and configure it to run at boot

```bash
sudo systemctl start mongod
```
```bash
sudo systemctl enable mongod
```
```bash
sudo systemctl status mongod
```
![Start Mongodb](<./images/Screenshot 2026-08-12 015646.png>)

### Add request-body parsing support

The `body-parser` package is used here to parse JSON data received in HTTP requests.

```bash
sudo npm install body-parser
```
![Install body-parser](<./images/Screenshot 2026-08-12 020933.png>)

### Create the application workspace

```bash
mkdir Books && cd Books
```

Initialize the Node.js project
```bash
npm init
```
![init folder](<./images/Screenshot 2026-08-12 021058.png.>)

Create the main server file
```bash
vim server.js
```
The server file contains the application bootstrap logic: MongoDB connection, JSON middleware, static-file handling, route registration, and the listener on port 3300.

```bash
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose'); // Make sure mongoose is installed and required
const path = require('path'); // To handle static file serving
const app = express();

// Connect to MongoDB
mongoose.connect('mongodb://localhost:27017/test', { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Middleware
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

// Routes
require('./apps/routes')(app);

// Start the server
app.set('port', 3300);
app.listen(app.get('port'), () => {
  console.log('Server up: http://localhost:' + app.get('port'));
});
```
![Server.js](<./images/Screenshot 2026-08-14 125008.png>)


## 4. Build the Express API and Data Model

Express was used to pass book information to and from our MongoDB database.
Mongoose package provides a straightforward schema-based solution to model the application data. Mongoose was used to establish a schema for the database to store data of the book register.

### Install the backend dependencies

```bash
sudo npm install express mongoose
```
![Install express](<./images/Screenshot 2026-08-14 125832.png>)

### Create the API module

```bash
mkdir apps && cd apps
```
Add `routes.js`

```bash
vim routes.js
```

The router implements the four main book operations:

```bash
const Book = require('./models/book');
const path = require('path');

module.exports = function(app) {
  // Get all books
  app.get('/book', async (req, res) => {
    try {
      const books = await Book.find({});
      res.json(books);
    } catch (err) {
      console.error(err);
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // Add a new book
  app.post('/book', async (req, res) => {
    try {
      const book = new Book({
        name: req.body.name,
        isbn: req.body.isbn,
        author: req.body.author,
        pages: req.body.pages
      });
      const result = await book.save();
      res.json({
        message: "Successfully added book",
        book: result
      });
    } catch (err) {
      console.error(err);
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // Update a book
  app.put('/book/:isbn', async (req, res) => {
    try {
      const updatedBook = await Book.findOneAndUpdate(
        { isbn: req.params.isbn },
        req.body,
        { new: true }
      );
      if (!updatedBook) {
        return res.status(404).json({ error: 'Book not found' });
      }
      res.json({
        message: "Successfully updated the book",
        book: updatedBook
      });
    } catch (err) {
      console.error(err);
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // Delete a book
  app.delete('/book/:isbn', async (req, res) => {
    try {
      const result = await Book.findOneAndRemove({ isbn: req.params.isbn });
      if (!result) {
        return res.status(404).json({ error: 'Book not found' });
      }
      res.json({
        message: "Successfully deleted the book",
        book: result
      });
    } catch (err) {
      console.error(err);
      res.status(500).json({ error: 'Internal Server Error' });
    }
  });

  // Serve static files
  // CHANGED: Using the '/*splat' syntax required by path-to-regexp v8
  app.get('/*splat', (req, res) => {
    res.sendFile(path.join(__dirname, '../public', 'index.html'));
  });
};
```
![Routes](<./images/Screenshot 2026-08-14 130017.png>)

### Add the Mongoose model

```bash
mkdir models && cd models
```
Create `book.js` inside the models directory

```bash
vim book.js
```
![](<./images/Screenshot 2026-08-14 130017.png>)

The schema below defines the fields required for every book document and makes the ISBN unique.

```bash
const mongoose = require('mongoose');

const bookSchema = new mongoose.Schema({
  name: { type: String, required: true },
  isbn: { type: String, required: true, unique: true },
  author: { type: String, required: true },
  pages: { type: Number, required: true }
});

module.exports = mongoose.model('Book', bookSchema);
```
![books](<./images/Screenshot 2026-08-14 130433.png>)


## 5. Connect the AngularJS Interface

The browser interface uses AngularJS and its `$http` service to communicate with the Express API. The controller loads existing books and provides functions for creating, updating, and deleting records.

### Create the public client files

```bash
cd ../..

mkdir public && cd public
```
Create `script.js` for the AngularJS controller

```bash
vim script.js
```
The controller below performs the API calls and keeps the browser's book list synchronized with the server.

```bash
var app = angular.module('myApp', []);

app.controller('myCtrl', function($scope, $http) {
  // Get all books
  function getAllBooks() {
    $http({
      method: 'GET',
      url: '/book'
    }).then(function successCallback(response) {
      $scope.books = response.data;
    }, function errorCallback(response) {
      console.log('Error: ' + response.data);
    });
  }

  // Initial load of books
  getAllBooks();

  // Add a new book
  $scope.add_book = function() {
    var body = {
      name: $scope.Name,
      isbn: $scope.Isbn,
      author: $scope.Author,
      pages: $scope.Pages
    };
    $http({
      method: 'POST',
      url: '/book',
      data: body
    }).then(function successCallback(response) {
      console.log(response.data);
      getAllBooks();  // Refresh the book list
      // Clear the input fields
      $scope.Name = '';
      $scope.Isbn = '';
      $scope.Author = '';
      $scope.Pages = '';
    }, function errorCallback(response) {
      console.log('Error: ' + response.data);
    });
  };

  // Update a book
  $scope.update_book = function(book) {
    var body = {
      name: book.name,
      isbn: book.isbn,
      author: book.author,
      pages: book.pages
    };
    $http({
      method: 'PUT',
      url: '/book/' + book.isbn,
      data: body
    }).then(function successCallback(response) {
      console.log(response.data);
      getAllBooks();  // Refresh the book list
    }, function errorCallback(response) {
      console.log('Error: ' + response.data);
    });
  };

  // Delete a book
  $scope.delete_book = function(isbn) {
    $http({
      method: 'DELETE',
      url: '/book/' + isbn
    }).then(function successCallback(response) {
      console.log(response.data);
      getAllBooks();  // Refresh the book list
    }, function errorCallback(response) {
      console.log('Error: ' + response.data);
    });
  };
});
```
![Script.js](<./images/Screenshot 2026-08-14 133914.png>)


### Build the HTML interface

```bash
vim index.html
```
The page defines the form used to enter book details and the table used to display the records returned by the API.

```bash
<!DOCTYPE html>
<html ng-app="myApp" ng-controller="myCtrl">
<head>
  <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.4/angular.min.js"></script>
  <script src="script.js"></script>
  <style>
    /* Add your custom CSS styles here */
  </style>
</head>
<body>
  <div>
    <table>
      <tr>
        <td>Name:</td>
        <td><input type="text" ng-model="Name"></td>
      </tr>
      <tr>
        <td>Isbn:</td>
        <td><input type="text" ng-model="Isbn"></td>
      </tr>
      <tr>
        <td>Author:</td>
        <td><input type="text" ng-model="Author"></td>
      </tr>
      <tr>
        <td>Pages:</td>
        <td><input type="number" ng-model="Pages"></td>
      </tr>
    </table>
    <button ng-click="add_book()">Add</button>
    <div ng-if="successMessage">{{ successMessage }}</div>
    <div ng-if="errorMessage">{{ errorMessage }}</div>
  </div>
  <hr>
  <div>
    <table>
      <tr>
        <th>Name</th>
        <th>Isbn</th>
        <th>Author</th>
        <th>Page</th>
        <th>Action</th>
      </tr>
      <tr ng-repeat="book in books">
        <td>{{ book.name }}</td>
        <td>{{ book.isbn }}</td>
        <td>{{ book.author }}</td>
        <td>{{ book.pages }}</td>
        <td><button ng-click="del_book(book)">Delete</button></td>
      </tr>
    </table>
  </div>
</body>
</html>
```
![HTML](<./images/Screenshot 2026-08-14 134455.png>)

### Launch the application

```bash
cd ..
```
```bash
node server.js
```
![Run server](<./images/Screenshot 2026-08-14 140112.png>)

Once `server.js` is running, the application listens on port `3300`. A second SSH session can be used to test the API locally with `curl` if required.

The Book Register can then be opened through a browser using the EC2 public IP address or public DNS name.

![Book register](<./images/Screenshot 2026-08-14 140449.png>)

Test the application by adding additional records

![Book register](<./images/Screenshot 2026-08-14 140715.png>)

Inspect the API's JSON response

![Book register](<./images/Screenshot 2026-08-14 140837.png>)

## 6. Result

The completed Book Register demonstrates the main flow of a MEAN-style application: AngularJS handles the browser interface, Express exposes the API, Node.js runs the server, and MongoDB persists the book data.

Using JavaScript across the application reduces the number of languages involved in the stack and gives the front end and back end a consistent development environment.
