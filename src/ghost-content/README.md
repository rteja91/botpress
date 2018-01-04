# Ghost Content Design Document

Ghost Content is the term referencing the generic approach implemented for managing different types of content, including content types (forms), flows, and potentially others.

## The problem

There are certain parts of the bot code that may change more often than the bot's code.
For such parts it's often common for less technical members of the team, or for engineers who simply prefer faster feedback cycles, to use the GUI tools to manage the content and observe the behaviour change on the live bot.
At the moment Botpress provides two such features:
* Content Manager
* Flow Editor

Both let you change the files that belong to the bot code through the help of the UI in the Botpress admin panel.

This all works good before you want to deploy the new version of your bot (including some code changes made by the developers).

It opens two problems:
1) certain hosting platforms (including Heroku, or Google Cloud, or any Docker-based deployment pipeline) will package the new version of your code as a comleteley new container. Which means all of your changes to the local files is gone as soon as you make the new successful deployment (the old container is shut down and thrown away)
2) What if your updated version of the code has some changes to the same content files that have been edited on the server through the UI.

Ghost Content feature is designed to answer both questions.

## Feature overview

With the introduction of the feature the botpress server behaves like that:
* all changes are not saved to the files anymore, instead they go to the database. This DB is referred to as the ghost version of the content from the files
* each change gets its unique random UUID recorded in the DB
* the repository also contains the list of UUIDs (in a file) indicating which of these changes were already incorporated back into the codebase (see below how)
* all parts that need to read from the content files now _always_ read the ghost version from the DB
* upon server start the following actions are taken:
  * the server finds all the revisions IDs still in the DB
  * it parses the UUIDs list from the bot code and deletes from the DB the IDs that are already in that file
  * if there are any IDs remaining in the DB the server generates persisted alert shown in the Botpress admin panel instructing the user to fetch the ghost content and merge it into the source code of their bot (see below how)
  * otherwise (if all the revisions are said to be reflected in the sources already) the server reads the corresponding files and saves their content to the ghost content DB (which makes it effectively available to the running application)
* how the ghost content can be incorporated back into the bot's source code:
  * the main secenario is using the botpress CLI, it will get the package from the server API, and add all the IDs to the UUIDs list, dump the new content into its place in the bot source code directory
  * alternatively the UI may eventually be present with download links, but it's not likely to happen in the first order

## Programmatic API

The Ghost Manager single instance is created in the `src/botpress.js` and should be passed to other parts of the app from there.

The menager operates on the level of **root folders**. For each _type_ of the files we want it to track it we should need to call `addFolder` for the main entry point of this content type. For example, the standard root folder of the content is `forms_data`, and the standard entry point of flows is `flows`.

> Please note that we're speaking about the topmost folder for the given content type. For example, the flows can be stored in nested folders under this folder, it's OK and you _don't need to_ configure Ghost Manager for each of the nested folders.

Below is the full Ghost Manager API, please keep it up to date.

> Also note, depending on the configuration the Ghost Manager can act in _real_ or mocked, _transparent_ mode. The downstream code should never care about the specifics and should treat the Ghost Manager as the black box conforming to the above explanation and the API listed below.

### addRootFolder(rootFolder: string, filesGlob: string) => Promise<void>

You need to call this method **once** for each use-case (like it's called once in the Content Manager).

The `rootFolder` is the entry point under which all the files we care about (in this specifc use-case) are located. The files can be nested arbitrary. The folder is _relative to the bot root directory_, also know as `projectLocation` (the directory where the `botfile.js` is located). The folder must exist before calling this method.

The `filesGlob` is a [glob](https://www.npmjs.com/package/glob) to indicate which files we want to be tracked by the Ghost Manager. Only files matching this glob will be affected.

Example:
```js
// ensure the folder exists
mkdirp.sync(formDataDir)
// track any *.json files in it (maybe nested)
await ghostManager.addRootFolder(formDataDir, '**/*.json')
```

This method call reconciles the ghost content DB table in accordance with the `$rootFolder/.ghost-revisions` file, and if the folder is in the _clean state_ (no revisions in the DB that are not listed in the `.ghost-revisions` files) it updates the DB with the fresh files content from the file system.

### recordRevision(rootFolder: string, file: string, content: string) => Promise<void>

Use this method whenever you want to store the content for the given file.

`rootFolder` has the same meaning as above.

`file` is the path to the file _relative to the_ `rootFolder`.

`content` is the updated content to b stored.

The method automatically records the revision ID for this operation which you normally don't have to know about.

Example:
```js
// record the new content for the `trivia_questions.json` file inside of the `formDataDir` folder
await ghostManager.recordRevision(formDataDir, 'trivia_questions.json', JSON.stringify(triviaQuestions, null, 2))
```

> Note that you should record the JSON files in indented form (the `null, 2`) params ensure that to simplify the git diffs.

> Note that the `file` param is not normalized, it's your responsibility to refer to the same file consistently across all calls to `recordRevision` and `readFile`.

### readFile(rootFolder: string, file: string) => Promise<string>

`rootFolder` and `file` have the same meaning as above.

Resolves with the content recorded for `file` in the most recent call to `recordRevision` (or the original file content from the disk, recorded on `addRootFolder` call). By design there's no way to read older revisions content (this may change in the future).

The promise will be rejected if the file's content cannot be found (which in the clean state is effectively equivalent to the fact the file `file` does not exist in `folder`)

### getPending => object

Returns the summary of all the files that have pending revisions (revisions not found in the corresponding `.ghost-revisions`).

The returned object has the following structure:

```js
{
  "forms_data": [
    { file: 'trivia_questions.json', id: 111, revision: '123-def', created_on: '2017-12-12T23:10:55Z', created_by: 'admin'},
    { file: 'trivia_questions.json', id: 112, revision: '456-def', created_on: '2017-12-12T23:10:45Z', created_by: 'admin'},
    { file: 'trivia_questions.json', id: 113, revision: '789-def', created_on: '2017-12-12T23:10:35Z', created_by: 'admin'},
    { file: 'greetings.json', id: 114, revision: 'abc-def', created_on: '2017-12-12T23:10:25Z', created_by: 'admin'},
  ],
  "another_folder": [...]
}
```

### getPendingWithContent => Promise<object>

Returns the data similar to the previous method, but with files contents. The format is different though:

```js
{
  "forms_data": {
    revisions: [
      '123-def',
      '456-def',
      '789-def',
      'abc-def'
    ],
    files: [
      { file: 'trivia_questions.json', content: '<FILE CONTENT>' },
      { file: 'greetings.json', content: '<FILE CONTENT>' }
    ]
  },
  "another_folder": { /* ... */ }
}
```