# Directus

Directus is an Open Data Platform built to democratize the database.
This platform provides everyone on your team, regardless of technical skill, equal access to data and digital file asset management, for any data model or project. First, link Directus to your desired SQL database and file storage adapter. After that, Directus enables you to perform CRUD operations, create users, assign roles with fully configurable permissions, build complex and granular queries, configure event-driven webhooks and task automation... the list goes on!

# Quickstart Guide

This quickstart guide is designed to get you up and running with a Directus Project in a few minutes. Along the way, you will better understand what Directus is, setup your Directus project locally or with Directus Cloud, and get a hands-on introduction to the App and API.
Follow the steps in this link to create your project.
https://docs.directus.io/getting-started/quickstart.html#quickstart-guide

# Adding Custom Logic

You can add custom logic to your project using two methods:

- Flows

  - Flows enable custom, event-driven data processing and task automation within Directus. Each flow is composed of one trigger, followed by a series of operations. You can find more about flows and how to use it in this link https://docs.directus.io/app/flows.html

- Extensions

  - Extensions provide a way to build, modify or expand Directus' functionality beyond the default, for your specific needs. There are three main categories of extensions:

  1. App Extensions (frontend extensions)
  2. API Extensions (backend extensions)
  3. Hybrid Extensions
     You can find more about extensions at this link: https://docs.directus.io/extensions/introduction.html

# Explaining how to add an extension to your project by a usecase
Our usecase is a project that have two collection types module and task
  - module collection has the following fileds:

    - name which is string
    - currentOrder which is number
    - status which is enumeration that has three values ( toDo, inProgress, done)

  - the task collection has the following fields:
    - name which is string
    - status which is boolean
    - moduleId which is relation with module collection

we need to add custom logic that enables us do the following:

  - if all tasks related to a module is false the module status is toDo
  - if one of the tasks related to a module is true the module status changes to inProgress
  - if all tasks related to a module is true the module status changes to done
  - we can't update a task status to true if a module with higher priority than its module is not done

## Solution

  ### 1. Create a Docker Compose File
Create a new empty folder on your Desktop called directus. Within this new folder, create the three empty folders database, uploads, and extensions.
Open a text editor such as Visual Studio Code, Copy and paste the following and save the file as docker-compose.yml

```
version: "3"
services:
  directus:
    image: directus/directus:10.8.3
    ports:
      - 8055:8055
    volumes:
      - ./database:/directus/database
      - ./uploads:/directus/uploads
      - ./extensions:/directus/extensions
    environment:
      KEY: "replace-with-random-value"
      SECRET: "replace-with-random-value"
      ADMIN_EMAIL: "admin@example.com"
      ADMIN_PASSWORD: "d1r3ctu5"
      DB_CLIENT: "sqlite3"
      DB_FILENAME: "/directus/database/data.db"
      WEBSOCKETS_ENABLED: true
```

Save the file.
Let's step through it:

- This file defines a single Docker container that will use the specified version of the directus/directus image.
- The ports list maps internal port 8055 is made available to our machine using the same port number, meaning we can access it from our computer's browser.
- The volumes section maps internal directus/database and directus/uploads to our local file system alongside the docker-compose.yml - meaning data is backed up outside of Docker containers.
- The environment section contains any configuration variables we wish to set.
  - KEY and SECRET are required and should be long random values. KEY is used for telemetry and health tracking, and SECRET is used to sign access tokens.
  - ADMIN_EMAIL and ADMIN_PASSWORD is the initial admin user credentials on first launch.
  - DB_CLIENT and DB_FILENAME are defining the connection to your database.
  - WEBSOCKETS_ENABLED is not required, but enables Directus Realtime.
 
Move to the directus directory and Run 
```shell
docker compose up
```

### 2. Setup project collections
Login using the admin credentials and create the module and task collections and its fields.

### 3. Create a hook extension


#### 1. install dependencies
Open a console to your preferred working directory and initialize a new extension, which will create the boilerplate code for your display.

```shell
npx create-directus-extension@latest
```

A list of options will appear (choose hook), and type a name for your extension (for example, directus-hook-updateTaskStatus). For this guide, select JavaScript.

Now the boilerplate has been created, then open the directory in your code editor.

#### 2. Build the hook
The entrypoint of your hook is the index file inside the src/ folder of your extension package. It exports a register function to register one or more event listeners.

The register function receives an object containing the type-specific register functions as the first parameter:
- filter — Listen for a filter event
- action — Listen for an action event
- init — Listen for an init event
- schedule — Execute a function at certain points in time
- embed — Inject custom JavaScript or CSS within the Data Studio

The second parameter is a context object with the following properties:
- services — All API internal services
- database — Knex instance that is connected to the current database
- getSchema — Async function that reads the full available schema for use in services
- env — Parsed environment variables
- logger — Pino instance.
- emitter — Event emitter instance that can be used to emit custom events for other extensions.


#### Filter
Filter hooks act on the event's payload before the event is emitted. They allow you to check, modify, or cancel an event.

The filter register function receives two parameters:
- The event name
- A callback function that is executed whenever the event is emitted.

The callback function itself receives three parameters:
- The modifiable payload
- An event-specific meta object
- A context object

The context object has the following properties:
- database — The current database transaction
- schema — The current API schema in use
- accountability — Information about the current user

#### Action
Action hooks execute after a defined event and receive data related to the event. Use action hooks when you need to automate responses to CRUD events on items or server actions.

The action register function receives two parameters:
- The event name
- A callback function that is executed whenever the event is emitted.

The callback function itself receives two parameters:
- An event-specific meta object
- A context object

The context object has the following properties:
- database — The current database transaction
- schema — The current API schema in use
- accountability — Information about the current user

You can find more about the other hooks at this link: https://docs.directus.io/extensions/hooks.html

Back to our usecase, in our case we will create filter hook and action hook.
the filter hook will act as a middleware and do the needed validation before updating the task status.
the action hook will update the module status after the task status updated .

Open the index.js file inside the /src directory and start write your custom logic


```javascript
export default ({ filter, action }, { database }) => {
  filter("task.items.update", async (payload, context) => {
    if (payload.status === "true") {
      const task = await database("task").where("id", context.keys[0]);

      const module = await database("module").where("id", task.moduleId);

      const higherMouldes = await database("module")
        .where("currentOrder", "<", module[0].currentOrder)
        .andWhere("status", "!=", "done");

      if (higherMouldes.length > 0) {
        throw new Error(
          "You can't update this task because there are higher priority tasks"
        );
      }
    }
  });

  action("task.items.update", async (context) => {
    // get task
    const task = await database("task").where("id", context.keys[0]);

    // get module
    const module = await database("module").where("id", task[0].moduleId);

    // get all tasks from module

    const tasks = await database("task").where("moduleId", module[0].id);

    // calculate module status based on tasks status

    // check if all tasks are false
    const allFalse = tasks.every((task) => task.status === "false");

    // check if all tasks are true
    const allTrue = tasks.every((task) => task.status === "true");

    if (allFalse) {
      await database("module").where("id", module[0].id).update({
        status: "todo",
      });
    } else if (allTrue) {
      await database("module").where("id", module[0].id).update({
        status: "done",
      });
    } else {
      await database("module").where("id", module[0].id).update({
        status: "inProgress",
      });
    }
  });
};
```

Build the hook with the latest changes

```shell
npm run build
```

#### 3. Add Hook to Directus:
When Directus starts, it will look in the extensions directory for any subdirectory starting with directus-extension-, and attempt to load them.
in our case we will create a folder named directus-extension-updateTaskStatus in hooks directory and move the package.json file and the index.js file generated in the dist folder
Restart the container to load the extension.
