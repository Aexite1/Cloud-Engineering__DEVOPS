# Deploying a MERN To-Do Application on AWS


## Overview

The MERN stack combines several JavaScript technologies to build a complete web application. Each part is responsible for a different layer of the system:

**MongoDB:** A document-oriented NoSQL database used to persist the application's data.

**Express.js:** A Node.js framework used to create the server, routes, and API endpoints.

**React:** A front-end library used to build the browser interface from reusable components.

**Node.js:** The JavaScript runtime that executes the backend application outside the browser.__

This documentation walks through the deployment process, beginning with the AWS server and ending with a working browser-based To-Do application connected to MongoDB.


## 1. Prepare the AWS Environment

An Ubuntu 24.04 LTS EC2 instance was created in the `us-north-1c` region. The instance used the `t3.small` size for this implementation.
The instance size was selected for these practical reasons:

- **Memory:** The additional memory provides more headroom for a full-stack application than a smaller burstable instance.

- **Burst performance:** T3 instances are designed for workloads that periodically need more CPU capacity.

- **General capacity:** The larger instance provides a more suitable baseline for running the backend and frontend during the exercise.

![Lunch Instance](<./images/Screenshot 2026-08-01 141107.png>)
![Lunch Instance](<./images/Screenshot 2026-08-01 141405.png>)

An SSH key named `mern-key` was associated with the instance so that it could be accessed remotely over SSH on port 22.

The EC2 security group was configured to permit the ports required by the application:

- **80/TCP:** HTTP traffic.

- **443/TCP:** HTTPS traffic.

- **22/TCP:** SSH administration access.

- **5000/TCP:** Express backend access.

- **3000/TCP:** React development server access.

![Security Rules](<./images/Screenshot 2026-08-01 142043.png>)

The private key was given the required restrictive permissions before it was used to establish the SSH connection:
```bash
chmod 400 mern-key.pem
```
```bash
ssh -i "mern-key.pem" ubuntu@16.170.110.235
```
In this example, `ubuntu` is the SSH user and `16.170.110.235` is the instance's public IPv4 address. Use your own instance address rather than copying this example.

![Connect to instance](<./images/Screenshot 2026-08-01 142503.png>)


## 2. Configure the Backend

### Update the Ubuntu packages

```bash
sudo apt update && sudo apt upgrade -y
```
![Update Packages](<./images/Screenshot 2026-08-01 144159.png>)


### Install Node.js

```bash
sudo apt-get install nodejs -y && sudo apt install nodejs
```
![Install nodejs](<./images/Screenshot 2026-08-01 145155.png>)

![Install node.js](<./images/Screenshot 2026-08-01 145346.png>)

**Note:** Installing Node.js this way also provides npm, which is used to install JavaScript packages and manage the application's dependencies.

### Confirm Node.js and npm

```bash
node -v        // Gives the node version

npm -v        // Gives the node package manager version
```
![node-npm version](<./images/Screenshot 2026-08-01 145416.png>)


### Initialize the Application

Create a directory for the To-Do application, enter it, and initialize it as an npm project.
```bash
mkdir TODO && ls && cd TODO

npm init
```

![Project dir](<./images/Screenshot 2026-08-01 150248.png>)

`npm init` creates `package.json`, which records basic project metadata and the packages required by the application.
Accept the prompts with the defaults where appropriate and confirm the generated configuration.
![pakage.json](<./images/Screenshot 2026-08-01 150547.png>)

### Add Express.js

Express provides the HTTP server layer for the application. It makes it straightforward to define routes, process requests, and return responses.

Install the Express dependency:

```bash
npm install express
```
![Install express](<./images/Screenshot 2026-08-01 150824.png>)

Create the main server file and verify that it exists:

```bash
touch index.js && ls
```
Install `dotenv` so configuration values can be loaded from an environment file:
```bash
npm install dotenv
```
![dotenv](<./images/Screenshot 2026-08-01 151236.png>)

Open `index.js` and add the server configuration shown below.
```bash
vim index.js
```
Add the following server code:

```bash
const express = require('express');
require('dotenv').config();

const app = express();

const port = process.env.PORT || 5000;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use((req, res, next) => {
  res.send('Welcome to Express');
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```
![index.js code](<./images/Screenshot 2026-08-01 152139.png>)

__Note:__ Port 5000 have been specified to be used in the code. This was required later on the browser.

Start the backend from the directory containing `index.js`:
```bash
node index.js
```
![Start server](<./images/Screenshot 2026-08-01 152525.png>)

Because port 5000 was allowed in the EC2 security group, the service can be reached externally for testing.

Test the server by opening the instance's public IP together with port 5000:

```bash
http://16.170.110.235:5000
```
![Express page](<./images/Screenshot 2026-08-01 152709.png>)


## 3. Build the API Routes

The API needs three core operations:
- Add a new task
- Retrieve the existing tasks
- Remove a task

Each operation maps to an API endpoint and uses the appropriate HTTP method: `POST` for creation, `GET` for retrieval, and `DELETE` for removal.

These endpoints are grouped into a router so the application can expose the required To-Do operations cleanly.

Create a `routes` directory, add `api.js`, and use it as the API router.
```bash
mkdir routes && cd routes && touch api.js
```
![Router](<./images/Screenshot 2026-08-01 153318.png>)
![Router](<./images/Screenshot 2026-08-01 170317.png>)
Add the route definitions below:

```bash
const express = require('express');
const router = express.Router();

router.get('/todos', (req, res, next) => {

});

router.post('/todos', (req, res, next) => {

});

router.delete('/todos/:id', (req, res, next) => {

});

module.exports = router;
```
![Route](<./images/route.png>)


## 4. Define the MongoDB Model

A Mongoose model provides the application with a structured interface for working with MongoDB documents.

The schema specifies the fields that each To-Do document should contain and the model is then used to create and query those documents.

In this project, the schema acts as the application's data structure. Mongoose also supports features such as virtual properties, although the To-Do schema here only requires the task field.
Mongoose is used as the ODM layer between Node.js and MongoDB, making schema definition and database operations easier.

Return to the project root and install Mongoose:
```bash
npm install mongoose
```
![Mongoose](<./images/Screenshot 2026-08-01 170409.png>)

Create a `models` directory and add `todo.js` for the To-Do schema and model.

```bash
mkdir models && cd models && touch todo.js
```
![](<./images/Screenshot 2026-08-01 171007.png>)

Add the schema and model definition:

```bash
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

// Create schema for todo
const TodoSchema = new Schema({
  action: {
    type: String,
    required: [true, 'The todo text field is required']
  }
});

// Create model for todo
const Todo = mongoose.model('todo', TodoSchema);

module.exports = Todo;
```
![Schema](<./images/Screenshot 2026-08-01 171440.png>)

The API router can now be connected to the Mongoose model so that requests are backed by database operations.

Open `routes/api.js`, replace the temporary route handlers, and connect the endpoints to the `Todo` model.

```bash
vim api.js
```

Replace the router contents with the database-backed implementation below:
```bash
const express = require('express');
const router = express.Router();
const Todo = require('../models/todo');

router.get('/todos', (req, res, next) => {
  // This will return all the data, exposing only the id and action field to the client
  Todo.find({}, 'action')
    .then(data => res.json(data))
    .catch(next);
});

router.post('/todos', (req, res, next) => {
  if (req.body.action) {
    Todo.create(req.body)
      .then(data => res.json(data))
      .catch(next);
  } else {
    res.json({
      error: "The input field is empty"
    });
  }
});

router.delete('/todos/:id', (req, res, next) => {
  Todo.findOneAndDelete({"_id": req.params.id})
    .then(data => res.json(data))
    .catch(next);
});

module.exports = router;
```
![Router update](<./images/Screenshot 2026-08-03 091809.png>)


## 5. Connect MongoDB Atlas

MongoDB Atlas is used as the managed database service. The database cluster is hosted in an AWS region for this implementation.

Create the MongoDB cluster and configure the database for the application.

MongoDB cluster configuration

![DB overview](<./images/Screenshot 2026-08-03 154947.png>)

The AWS cloud provider and `us-north-1` region were selected for the database deployment.

![Cloud region](<./images/Screenshot 2026-08-03 155007.png>)

For this development exercise, network access was opened broadly to allow the EC2 host to connect. This is convenient for testing but should be restricted to trusted IPs or networks in a production deployment.

![Network access](<./images/Screenshot 2026-08-03 160558.png>)

A database named `todo_db` and a `todos` collection were created for the application data.

Create an `.env` file in the project root to hold the MongoDB connection value:
```bash
touch .env && vim .env
```
Add the MongoDB connection string as an environment variable:

```bash
DB = ‘mongodb+srv://<username>:<password>@<network-address>/<dbname>?retryWrites=true&w=majority’
```
![Connection string](<./images/Screenshot 2026-08-03 170819.png>)


Update `index.js` so the backend loads the connection string from the environment and establishes the MongoDB connection.

```bash
vim index.js
```

Replace the server file with the database-enabled version below:

```bash
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const routes = require('./routes/api');
const path = require('path');
require('dotenv').config();

const app = express();

const port = process.env.PORT || 5000;

// Connect to the database
mongoose.connect(process.env.DB, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log(`Database connected successfully`))
  .catch(err => console.log(err));

// Since mongoose promise is deprecated, we override it with Node's promise
mongoose.Promise = global.Promise;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use(bodyParser.json());

app.use('/api', routes);

app.use((err, req, res, next) => {
  console.log(err);
  next();
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});

```
![index.js](<./images/Screenshot 2026-08-03 165715.png>)

Keeping the database URI in `.env` separates secrets and environment-specific configuration from the application source code. The `.env` file should not be committed to a public repository.

Start the backend again:
```bash
node index.js
```
![Server](<./images/Screenshot 2026-08-03 170843.png>)

A deprecation warning appeared when the server started.
The warning was addressed by removing the options that were no longer required by the installed Mongoose/MongoDB driver version.


## 6. Test the Backend API

Postman was used to exercise the API independently of the React client.
For requests that require a request body, JSON containing the expected `action` field was supplied.

Open Postman and configure the request endpoint:

```bash
http://16.170.110.235:5000/api/todos
```
![Post request](<./images/Screenshot 2026-08-07 031839.png>)

### Test the POST Endpoint


![Post request](<./images/Screenshot 2026-08-07 133156.png>)
![Post request](<./images/Screenshot 2026-08-07 132514.png>)
![Post request](<./images/Screenshot 2026-08-07 133415.png>)



### Verify the Stored Documents

![DB Collections](<./images/Screenshot 2026-08-07 134931.png>)


### Test the GET Endpoint

The GET operation retrieves the current To-Do records from MongoDB and returns them to the client as JSON.

![Get request](<./images/Screenshot 2026-08-07 031839.png>)

### Test the DELETE Endpoint

![Delete request](<./images/Screenshot 2026-08-07 134859.png>)

### Query the Collection Again

![Get request](<./images/Screenshot 2026-08-07 134931.png>)


## 7. Build the React Frontend

The backend is now ready, so the next stage is to provide a browser interface that communicates with the API.

From the project root, generate the React client:

```bash
npx create-react-app client
```
![React](<./images/Screenshot 2026-08-07 141355.png>)

This creates a `client` directory containing the React application source and npx installs all that is needed for the app to run effectivly.

### Run the React Application

Before running the two parts of the application together, install the development dependencies used to manage the local workflow.

- **concurrently:** Runs the frontend and backend processes from one command.
```bash
npm install concurrently --save-dev
```
![Concurrent](<./images/Screenshot 2026-08-07 143713.png>)


- **nodemon:** Watches the backend files and restarts Node.js when changes are detected.
```bash
npm install nodemon --save-dev
```
![Nodemon](<./images/Screenshot 2026-08-07 143807.png>)


Update the root `package.json` scripts so both development processes can be started together:
```bash
"scripts": {
  "start": "node index.js",
  "start-watch": "nodemon index.js",
  "dev": "concurrently \"npm run start-watch\" \"cd client && npm start\""
}
```
![Package.json](<./images/Screenshot 2026-08-07 145150.png>)

### Configure the React API Proxy

Enter the React client directory:
```bash
cd client
```

Open the client's `package.json`:
```bash
vim package.json
```
![Package](<./images/Screenshot 2026-08-07 145150.png>)

Add the development proxy entry:
```bash
“proxy”: “http://localhost:5000”
```
![Proxy](<./images/Screenshot 2026-08-08 090431.png>)

The proxy allows the React development server to forward API requests to the Express server, so frontend code can call the API without hard-coding the backend host into every request.
http://localhost:5000 rather than always including the entire path like http://localhost:5000/api/todos

Return to the project root and launch both processes:
```bash
npm run dev
```
![](<./images/Screenshot 2026-08-08 090847.png>)
The React development server should now be available on port 3000.
![](<./images/Screenshot 2026-08-08 091803.png>)

**Note:** Port 3000 must be permitted by the EC2 security group if the development server is being accessed directly over the internet.


## 8. Create the React Components

React components divide the interface and application logic into smaller reusable units. For this To-Do application, the UI is separated into components for input, task listing, and the main To-Do container.

```bash
cd client
```

Enter the React `src` directory:
```bash
cd src
```

Create a `components` directory inside `src`:

```bash
mkdir components
```
Enter the components directory:
```bash
cd components
```
Create the three component files:
```bash
touch Input.js ListTodo.js Todo.js
```
![Component files](<./images/Screenshot 2026-08-10 125106.png>)

### Input Component
```bash
vim Input.js
```
Add the component implementation:

```
import React, { Component } from 'react';
import axios from 'axios';

class Input extends Component {
  state = {
    action: ""
  }

  handleChange = (event) => {
    this.setState({ action: event.target.value });
  }

  addTodo = () => {
    const task = { action: this.state.action };

    if (task.action && task.action.length > 0) {
      axios.post('/api/todos', task)
        .then(res => {
          if (res.data) {
            this.props.getTodos();
            this.setState({ action: "" });
          }
        })
        .catch(err => console.log(err));
    } else {
      console.log('Input field required');
    }
  }

  render() {
    let { action } = this.state;
    return (
      <div>
        <input type="text" onChange={this.handleChange} value={action} />
        <button onClick={this.addTodo}>add todo</button>
      </div>
    );
  }
}

export default Input;
```
![Input](<./images/Screenshot 2026-08-08 092313.png>)

The input component uses Axios to send HTTP requests. Install Axios from the client project before using it:

Return to the client project:
```bash
cd ../..
```
Install Axios:
```bash
npm install axios
```
![](<./images/Screenshot 2026-08-08 092656.png>)

Return to the components directory:
```bash
cd src/components
```

### ListTodo Component

```bash
vim ListTodo.js
```
Add the task-list component:

```bash
import React from 'react';

const ListTodo = ({ todos, deleteTodo }) => {
  return (
    <ul>
      {
        todos && todos.length > 0 ? (
          todos.map(todo => {
            return (
              <li key={todo._id} onClick={() => deleteTodo(todo._id)}>
                {todo.action}
              </li>
            );
          })
        ) : (
          <li>No todo(s) left</li>
        )
      }
    </ul>
  );
}

export default ListTodo;
```
![Todo list](<./images/Screenshot 2026-08-08 092943.png>)

### Todo Container

```bash
vim Todo.js
```

```bash
import React, { Component } from 'react';
import axios from 'axios';

import Input from './Input';
import ListTodo from './ListTodo';

class Todo extends Component {
  state = {
    todos: []
  }

  componentDidMount() {
    this.getTodos();
  }

  getTodos = () => {
    axios.get('/api/todos')
      .then(res => {
        if (res.data) {
          this.setState({
            todos: res.data
          });
        }
      })
      .catch(err => console.log(err));
  }

  deleteTodo = (id) => {
    axios.delete(`/api/todos/${id}`)
      .then(res => {
        if (res.data) {
          this.getTodos();
        }
      })
      .catch(err => console.log(err));
  }

  render() {
    let { todos } = this.state;
    return (
      <div>
        <h1>My Todo(s)</h1>
        <Input getTodos={this.getTodos} />
        <ListTodo todos={todos} deleteTodo={this.deleteTodo} />
      </div>
    );
  }
}

export default Todo;
```
![Todo](<./images/Screenshot 2026-08-08 093116.png>)

Update the default React application by removing the starter content and rendering the To-Do component.

Return to `src`:
```bash
cd ..
```

Ensure to be in the src folder and run:
```bash
vim App.js
```

Replace the contents of `App.js` with:

```bash
import React from 'react';
import Todo from './components/Todo';
import './App.css';

const App = () => {
  return (
    <div className="App">
      <Todo />
    </div>
  );
}

export default App;

```
![Appjs](<./images/Screenshot 2026-08-08 093620.png>)

### Style the Application

```bash
vim App.css
```

Use the following CSS to style the application layout, input, button, and task list:

```bash
.App {
  text-align: center;
  font-size: calc(10px + 2vmin);
  width: 60%;
  margin-left: auto;
  margin-right: auto;
}

input {
  height: 40px;
  width: 50%;
  border: none;
  border-bottom: 2px #101113 solid;
  background: none;
  font-size: 1.5rem;
  color: #787a80;
}

input:focus {
  outline: none;
}

button {
  width: 25%;
  height: 45px;
  border: none;
  margin-left: 10px;
  font-size: 25px;
  background: #101113;
  border-radius: 5px;
  color: #787a80;
  cursor: pointer;
}

button:focus {
  outline: none;
}

ul {
  list-style: none;
  text-align: left;
  padding: 15px;
  background: #171a1f;
  border-radius: 5px;
}

li {
  padding: 15px;
  font-size: 1.5rem;
  margin-bottom: 15px;
  background: #282c34;
  border-radius: 5px;
  overflow-wrap: break-word;
  cursor: pointer;
}

@media only screen and (min-width: 300px) {
  .App {
    width: 80%;
  }

  input {
    width: 100%;
  }

  button {
    width: 100%;
    margin-top: 15px;
    margin-left: 0;
  }
}

@media only screen and (min-width: 640px) {
  .App {
    width: 60%;
  }

  input {
    width: 50%;
  }

  button {
    width: 30%;
    margin-left: 10px;
    margin-top: 0;
  }
}

```
![App css](<./images/Screenshot 2026-08-08 094101.png>)

### Set the Global Styles

```bash
vim index.css
```
![src](<./images/Screenshot 2026-08-08 094118.png>)


Replace the global stylesheet with:

```bash
body {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto", "Oxygen", "Ubuntu", "Cantarell", "Fira Sans", "Droid Sans", "Helvetica Neue", sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  box-sizing: border-box;
  background-color: #282c34;
  color: #787a80;
}

code {
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New", monospace;
}
```
![index css](<./images/Screenshot 2026-08-08 094426.png>)

Return to the project root:
```bash
cd ../..
```

Run:
```bash
npm run dev
```
![dev](<./images/Screenshot 2026-08-08 103107.png>)

At this stage, the MERN application can add tasks, retrieve the saved list, and remove tasks through the React interface.

Open the React application in a browser:

![Todo web](<./images/Screenshot 2026-08-08 104802.png>)

Create a few sample tasks from the browser:

![Todos](<./images/Screenshot 2026-08-10 133526.png>)

Finally, confirm that the submitted tasks appear in MongoDB.

![Todo](<./images/Screenshot 2026-08-10 133645.png>)

## Conclusion

The implementation demonstrates how MongoDB, Express, React, and Node.js can be combined into a functional full-stack application and hosted on an AWS EC2 environment. The process also covers API creation, database integration, frontend development, and basic end-to-end testing.
