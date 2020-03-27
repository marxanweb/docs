# Developer Documentation
* Will be replaced with the ToC, excluding the "Contents" header
{:toc}  

[Back to documentation](index.html)

The Developer Documentation is aimed at software developers who want to: extend Marxan Web with new features; create new websites from the data or want to use desktop GIS tools to link directly to the Marxan Web database. Each of these topics is described below.  

## Overview of extending Marxan Web
Marxan Web is a piece of client-server software with the client running in a Web Browser (marxan-client) and the server running cross-platform on any operating system (marxan-server). The high level architecture is a loosely-coupled application that uses REST Services and WebSockets to communicate between the marxan-client and marxan-server. This loosely coupled system means that from the developers point of view the system is open and can be extended quickly and easily using the marxan-server API.  

Components on marxan-server include the PostGIS spatial database, the Tornado Web Server, the DOS-based Marxan software and the marxan-server custom software written in Python. The components of marxan-client are a web application written in React, the MapboxGL Javascript Mapping Library and a set of Vector Tile services to provide the mapping layers. These are all open-source technologies and can be customised and extended without restrictions. 

To create new features in Marxan Web it is necessary to create new user interface components in the marxan-client and then (if necessary) to provide the necessary services in marxan-server to either get/set any user data or undertake some kind of analysis or processing. Once these new features have been developed and tested they can be pulled back into the repo as part of the core Marxan Web and updated wherever they have been installed.  For more information see the [Administrator Documentation - Updates](admin.html#updates).  

It is not necessary to create new user interface components in marxan-client if you want to develop completely new applications for the web. These new tools can use the same underlying marxan-server API to interact with the storage and processing, but within an entirely new application. In these cases it may be necessary to authenticate with the marxan-server using secure cookie authentication - for more information see [Authentication](#authentication).  

### Architecture
#### Understanding the architecture
As described above Marxan Web is a client-server piece of software with the client running in a browser and the following figure shows the high-level architecture of marxan-server. Each of the components in this architecture are described in the [marxan-server](#marxan-server) section.  

<img src='images/admin_marxan_server.png' title='Marxan Web Architecture' class='docsImage'>

#### Typical workflow
This following is a typical simplified workflow for a running Marxan Web application:

- The marxan-server (marxan-server.py) is started [Adminstrator Documentation - Starting/stopping marxan-server](admin.html#startingstopping-marxan-server)
- The user opens the marxan-client in a web browser and logs in - for more information see [Authentication](#authentication).  
- REST requests are sent to the marxan-server API as GET, POST or WebSocket requests.  
- The marxan-server matches the REST endpoint to an internal class in the marxan-server.py file, runs the code and returns the results
- Returned json data is bound to user interface components using the React framework

### Technologies
#### marxan-server
marxan-server uses the Tornado Web Server for communication between the marxan-client and marxan-server. Tornado is a lightweight cross-platform server that supports SSL, WebSockets and server extensions and allows you to define endpoints that map a REST url endpoint to an internal Python class. For more information see [Creating REST services](#creating-rest-services). The implementation of these REST endpoints is in the marxan-server.py file in the marxan-server folder and when you run this file you are starting marxan-server.    

The PostGIS database is used to manage all of the spatial data that is created in Marxan Web and is the main processing engine for any intersections or other analyses. This fully-featured database means that any future spatial requirements can be met easily with the existing bundled database. Within marxan-server the connections to the database are managed using the psycopg2 Python library and the connection settings are set in the server.dat file. For more information see [Administrator Documentation - Database configuration](admin.html#database-configuration) and [Interacting with PostGIS](#interacting-with-postgis).  

The DOS-based Marxan software is still the heart of Marxan Web and it is this software that does the systematic conservation planning. All of the various *.dat files that are used in the DOS-version are managed in the marxan-client and marxan-server - for more information see [Migration Guide - Where are all the .dat files?](migration.html#where-are-all-the-dat-files).  

#### marxan-client
The marxan-client uses the React Framework which is a Javascript Framework developed by Facebook that is used to build user-interfaces where UI components use data binding to bind the json data coming from REST calls. 

Another technology that is used in the marxan-client software is Mapbox mapping technology. This technology offers a number of benefits:

- Performance is excellent, even with hundreds of thousands of planning units
- Maps can be restyled on the fly 
- Maps can be rotated and tilted to get the best viewing perspective

The most important of these is the performance: the MapboxGL technology means that the non-spatial results that Marxan produces (i.e. a csv file with hundreds of thousands of rows with lots of numbers in) can be mapped on-the-fly in the client. This cannot be achieved with conventional web GIS. 

Finally, the mapping data itself is delivered through Vector Tiles that are sourced from Mapbox and ESRI (and other providers in the future). These Vector Tiles are based on OpenStreetMap data and all of the feature attributes are sent to the browser with the spatial data meaning that they can be styled and queried on-the-fly in the browser.  

## marxan-server development
This section provides a quick guide to getting going in extending the marxan-server software.  

Firstly, in order to extend the marxan-server software you will need to fork the GitHub repo to be able to make changes locally and then submit them back to the main repo (through pull requests). To fork the repo in GitHub, goto the [repo](https://github.com/marxanweb/marxan-server) and follow the instructions [here](https://help.github.com/en/articles/fork-a-repo).  You will also need to install all of the prerequisites (e.g. Python libraries and PostGIS) - for more information see the installation instructions for the [marxan-server](https://github.com/marxanweb/marxan-server) repo.  

Now that repo is forked you can edit the marxan-server.py file to add your own extensions using your favorite Integrated Development Environment (IDE). Methods for extending marxan-server are described in the following sections.  

### Creating REST services
The following section in the marxan-server.py file is used to map between REST endpoints and Python classes that implement that feature. So, for example in the code below the url which ends in /marxan-server/testTornado will call the testTornado class and use that class to return the data to the client. It's as simple as that! Any number of new REST endpoints can be added to this list in order to create new features in marxan-server. 

```
def make_app():
    return tornado.web.Application([
        ("/marxan-server/testTornado", testTornado),
        ..
        ..
        
class testTornado(MarxanRESTHandler):
    def get(self):
        self.send_response({'info': "Tornado running"})

```

REST services are ideal if the execution time is short and communication between marxan-server and marxan-client is just a request and response. Longer running or more complex services should be creating using WebSockets - for more information see [Creating WebSocket extensions](#creating-websocket-extensions).  

#### Controlling access to REST services
There is one important additional step in creating new features through REST services - authorising those services. For all REST services, access is controlled through the use of roles (for more information see the [User Guide - Roles](user.html#roles)) and in marxan-server the mapping between which roles have access to which services is controlled through the ROLE_UNAUTHORISED_METHODS dictionary at the top of the marxan-server.py file. A curtailed example of this dictionary is shown below:

```
ROLE_UNAUTHORISED_METHODS = {
    "ReadOnly": ["createProject","createImportProject".. etc],
    "User": ["testRoleAuthorisation","deleteProject".. etc],
    "Admin": []
}

```

The ROLE_UNAUTHORISED_METHODS dictionary has an entry for each role and for each role it has a list of the methods that that role is NOT ALLOWED to access. So, in the example above, the ReadOnly role is not allowed to access the createProject service or many others. The admin role can access any service.  

When you have finished developing your new REST service, make sure that you control access to that service using the ROLE_UNAUTHORISED_METHODS dictionary and restart marxan-server. For more information see [Adminstrator Documentation - Starting/stopping marxan-server](admin.html#startingstopping-marxan-server). 

The marxan-client uses the information in the ROLE_UNAUTHORISED_METHODS dictionary firstly to show/hide relevant user interface components and secondly to physically stop any unauthorised access to a service based on the currently logged on users role.  

#### Testing new REST services
When you are in the process of developing new REST services it is more convenient not to have to worry about controlled access to services and authentication and just to be able to get on with developing and testing those new services. To do this, set the DISABLE_SECURITY to True and this disables all security (including CORS restrictions). At the end of the development process, make sure that you control access properly to the service.  

### Interacting with PostGIS
PostGIS is used by marxan-server to manage all features and planning grids and there are a set of methods for creating and deleting those entities. All new features and planning units that are created in Marxan Web will be created as tables in the marxan schema with a globally unique identifier prefixed with the following letters (these tables will be stored in the WGS84 coordinate system, EPSG:4326):  

- f_ - Features  
- fs_ - Features that were imported and split on a field name  
- gbif_ - Features imported from GBIF  
- pu_ - Planning units  

In addition, new features and planning grids will each have a respective new record in the 'metadata_planning_units' or 'metadata_interest_features' tables. These tables are used to capture the metadata for these entities.  

There are three other tables in PostGIS that are used by marxan-server to create new planning grids (either marine or terrestrial): 

- eez_simplified_1km - Contains the marine extent of all countries in the world  
- gaul_2015_simplified_1km - Contains the country boundaries of the world
- gaul_eez_dissolved - Contains the union of the marine and terrestrial extents for all countries in the world

To help interacting with the PostGIS database, you can use the PostGIS class which provides some convenience methods for executing queries, getting data frames, importing shapefiles, creating indices etc. This class is available through the singleton object called pg which maintains a pool of asynchronous database connections.   

### Interacting with Mapbox
All new features and planning grids that are created in marxan-server have to be uploaded to Mapbox to enable visualisation on the map - and the unique identifiers created in marxan-server are used to uniquely identify those tilesets in Mapbox. There are some convenience classes and methods for interacting with Mapbox in marxan-server including: _uploadTilesetToMapbox, _uploadTileset and _deleteTileset.  

### Creating WebSocket extensions
For most purposes in marxan-server, creating extensions by creating new REST services should be sufficient. However, for more long-running or complex extensions WebSockets should be used. WebSockets provide a mechanism for communication between marxan-client and marxan-server that is stateful and persistent so messages can be sent back and forth at regular intervals. The marxan-client uses WebSocket classes to update the Log tab in Marxan Web with information on the progress of long-running jobs, e.g. a Marxan run or the import of an existing Marxan project. 

To create WebSocket extensions, subclass the MarxanWebSocketHandler class. There are already a few example subclasses of the MarxanWebSocketHandler class:  

- runMarxan - For running Marxan jobs  
- QueryWebSocketHandler - For running PostGIS queries  
- importFeatures - For importing features  
- importGBIFData - For importing GBIF data  
- updateWDPA - For updating the World Database of Protected Areas  

Follow these example classes to create your own WebSocket extensions.  

## marxan-client development
The marxan-client software is developed using the React Framework from Facebook and was created using the create-react-app tool. This is a nodejs package that contains a development server, tools for rapidly creating user interfaces and compiling and building the code for  production. In order to develop and extend marxan-client, use the following steps:  

- Fork the [repo](https://github.com/marxanweb/marxan-client) and clone it to a development machine
- Change to the marxan-client folder and run npm install to install all of the nodejs dependencies (make sure you have the latest version of nodejs and npm installed)  

```
cd marxan-client
marxan-client/npm install
```

- You can now write React code to extend marxan-client and test it with the development server:

```
npm start
```
- When ever your source code changes this is detected and the web page will automatically reload.   

### Building 
Once you have finished your React development, you can build the repo using the npm run build command:  

```
npm run build
```

This will create a compiled and minified version of your code that can be deployed to production - the build folder will contain all the code you need for deployment.  

### Deploying
Your built application can be deployed simply by hosting the build folder on a suitable web server. If you want your new features to be available in other marxan-client deployments, then submit a pull request to the main marxan-client repo.  

The simplest way to deploy is to use host the built files on GitHub pages on GitHub. Follow the instructions [here](https://create-react-app.dev/docs/deployment/#github-pages) and then use the following command:  

```
npm run deploy
```


## Building new user interfaces on Marxan data
### marxan-server API
### Authentication

## Linking desktop GIS to the Marxan database
