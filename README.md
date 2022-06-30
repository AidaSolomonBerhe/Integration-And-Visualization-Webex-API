# Integration and Visualization with Webex API

This chapter is an optional and supplementary part of the thesis suggested by Telenor. 
Explanatory figures for the README.md file can be found at the following link under the folder "Main Document" https://drive.google.com/drive/folders/1SEbliwp19uP-DIDlsrI5AyyWwibCCCXh?usp=sharing. This folder contains the whole thesis pdf file. The figures can be found alongside documentation in Chapter 7 within the pdf file. Furthermore, the pdf file has an appendix containing the source code.



## Objectives

### General Objective

Through integration with Webex's API, Webex meeting quality is visualized. Successful execution and description convey a solid understanding of data visualization through REST API.

### Specific Objective
    - Create a web server (Integration server) as a test environment for the Integration. 
    - Register an Integration with Webex API 
    - Create scripts that retrieve meeting room quality in JSON format and save it
    - Visualize the meeting room experience in a creative manner

## Technology and Tools

This subchapter demonstrates the APIs (Application Program Interface), programming languages, and other technologies deployed to create the webserver (Integration server or application), integrate with Webex's API, and visualize Webex meeting quality.

### Webex API
Webex APIs facilitate access to the Webex Platform to develop Bots, Integrations, and Guest Issuer applications. Webex APIs enable direct access to the Cisco Webex Platform for one's application, allowing one to:
    - Create a Webex area and invite individuals

    - Search for members of one's organization

    - Post communications in a Webex area

    - Get Webex space history or receive real-time notifications when others publish new messages, and much more.

### REST API

REST stands for Representational State Transfer. REST API is an architecture that deploys an API using HTTP (Hypertext Transfer Protocol) requests to send and receive data. Resources entailed in a RESTful service can be accessed and manipulated using GET, POST, PUT, DELETE, and other HTTP methods. 

The transfer MIME type (Multipurpose Internet Mail Extensions) used in most cases is JSON. However, other MIME types can be used in data encoding. MIME types specify the nature and format of a document, file, or collection of bytes (https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types).

### Programming Languages
The program code is written in Golang {https://go.dev/}, HTML (HyperText Markup Language), and JavaScript. The database used is SQLite {https://www.sqlite.org/index.html}. The front end is rendered as a static site.  Go is chosen because of the language's simple syntax, which makes explaining code snippets uncomplicated.

The application is focused on fetching Meeting Analytics Quality data. This data is only accessible for one request every 5 minutes for each meeting data fetched. Therefore, having a cache in the application is a necessity. Thus, the application uses an SQLite database.

SQLite in the application allows for fault tolerance. When the client makes multiple requests for Meeting Analytics Quality data, any subsequent request that hits the '429 Too Many Request' status code is redirected to the cache, for the result is cached. SQLite is chosen because of the language's simplicity and requires little to no setup to be employed.

The application also uses cookies to store the client_secret, client_id, access_token, and refresh_token in the client’s browser. This data is necessary for the application to interact with the Webex API. Due to time-boundness and security measures, the data is not saved in the SQLite database. The client_secret, access_token, and refresh_token change over time.

### Integrations

Integrations are means by which one can request permission to invoke the Webex REST API on behalf of a different Webex user. In order to accomplish this safely, the API supports the OAuth 2 standard, which enables third-party integrations to obtain a temporary access token for authenticating API calls rather than requesting the user's password.

### OAuth

OAuth is an open standard for authentication usually employed by REST APIs to obtain scoped and limited access to the authorizing client’s data without giving the client’s password to the application. Webex OAuth flow demonstrates procedures to access protected resources under the scopes “analytics:read_all” and “meeting:schedules_read” in the Webex API. Scopes determine the level of access required for the integration.

The program requires that the client creates an integration with Webex. During the integration creation, the scopes meeting:schedules_read and analytics:read_all will be added as request parameters in the Authorization Request call. Permissions that the scopes grant include:

    - meeting:schedules\_read allows fetching all meetings that have been conducted for the account.
    - analytics:read\_all allows fetching all meeting room activity. This includes quality metrics.

On success, the client will be provided with the client_id and client_secret that will be used to initiate the OAuth Flow. If the scope of the application changes, the client_secret will also change, and the previous value is rendered invalid; In this event, the Webex Integration page for the account will have a new client_secret value.

The OAuth Flow is used to request access to client data/resources asked for through login for the client on the resource providers page (Webex Login Page). On successful login, the application server is triggered with a redirect. The redirect URI points to the application server, needs to be publicly accessible and able to accept the request from Webex. Using cloud services such as Amazon Web Services or Google Cloud to host the application code makes this possible. 

After successful login, the redirect request from the OAuth Flow comes with a code parameter that is used to fetch an access_token and refresh_token. To request this data, the client provides the code alongside the client_id and client_secret provided during the application integration creation. With this bit out of the way, the final response is a JSON object with the following fields:

    - The access_token is used to request any resources under the scope approved for in the Webex API. 
    - The refresh_token is used to request a new access_token and refresh_token after the current access_token has expired.

```json
{
    "access_token": "...",
    "expires_in": 0000,
    "refresh_token": "...",
    "refresh_token_expires_in": 0000,
}
```

### Accounts and Authentication

For a user to log into the webserver (application), the user is obliged to have a Cisco Webex account. Moreover, creating an Integration in Webex with the scopes "analytics:read_all" and "meeting:schedules_read" is mandatory. Doing so generates client_id and client_secret.

Another requirement is that the user needs to have a Cisco Webex account associated with a corporate/ enterprise network to access the Cisco Webex Control Hub. Without access to the Control Hub, getting analytics data and visualization of meeting room experience would be impossible. Telenor has granted access to the Control Hub via the tenant 'Telenor Hoyskolestudent 2022'.

The web server created for this specific task has the address http://3.222.86.122. 

## Implementation

### Crucial Listings
At the start of the application, the index page prompts the client to provide input with a form.

``` html
{
    <body>
   <h1>Start OAuth Flow</h1>
   <form method="POST" action="/init">
       <label for="client_id">Client ID</label><br />
       <input type="text" name="client_id" id="client_id"></br>
       <label for="client_secret">Client Secret</label></br>
       <input type="text" name="client_secret" id="client_secret"></br>
       <label for="scope"></label><br />
       <section>
           <h3>Scopes:</h3>
           <ul>
               <li>analytics:read_all</li>
               <li>meeting:schedules_read</li>
           </ul>
       </section>
       <input type="submit">
   </form>
</body>
}
```
Form submission will initiate the OAuth flow, alternatively granting access to the application for the client’s meeting data.

The Resource APIs consumed by the application are:
    
    - The List Meetings API is used to request all meetings available for the client. The HTTP response body is a JSON object. On success, a list of meetings conducted by the client is returned.
    - The Get Meeting Qualities API is used to request meeting quality data, provided a meeting_id is specified in the request parameters.

The response is transformed to a JSON object with the following array structure to allow for readability:

```json
{
     {
       "meeting_id": "some_id",
       "data_point": "audio_in",
       "start_time": "2022-05-02T17:54:19.852Z",
       "end_time": "2022-05-02T17:55:19.833Z",
       "packet_loss": [ 0, -1],
       "latency": [ 79, 79],
       "jitter": [ 0, -1]
   },
   {
       "meeting_id": "some_id",
       "data_point": "audio_out",
       "start_time": "2022-05-02T17:54:19.852Z",
       "end_time": "2022-05-02T17:55:19.833Z",
       "packet_loss": [ 0, 0],
       "latency": [ 79, 79],
       "jitter": [ 11, 13]
   },
   {
       "meeting_id": "some_id",
       "data_point": "video_in",
       "start_time": "2022-05-02T17:54:19.852Z",
       "end_time": "2022-05-02T17:55:19.833Z",
       "packet_loss": [ -1, -1],
       "latency": [ -1, -1],
       "jitter": [ -1, -1]
   },
   {
       "meeting_id": "some_id",
       "data_point": "video_out",
       "start_time": "2022-05-02T17:54:19.852Z",
       "end_time": "2022-05-02T17:55:19.833Z",
       "packet_loss": [ -1, -1],
       "latency": [ -1, -1],
       "jitter": [ -1, -1]
   },
   {
       "meeting_id": "some_id",
       "data_point": "share_in",
       "start_time": "2022-05-02T17:54:19.852Z",
       "end_time": "2022-05-02T17:55:19.833Z",
       "packet_loss": [ -1, -1],
       "latency": [ -1, -1],
       "jitter": [ -1, -1]
   },
   {
       "meeting_id": "some_id",
       "data_point": "share_out",
       "start_time": "2022-05-02T17:54:19.852Z",
       "end_time": "2022-05-02T17:55:19.833Z",
       "packet_loss": [ -1, -1],
       "latency": [ -1, -1],
       "jitter": [ -1, -1]
   }
}
```

## Visualization

### Elements of Visualization

The analytics visualization for the data uses three data points:
    
    - Audio In - The collection of downstream (sent to the client) audio quality data.
    - Audio Out - The collection of upstream (sent from the client) audio quality data.
 
    - Video In - The collection of downstream (sent to the client) video quality data.
    - Video Out  - The collection of upstream (sent from the client) video quality data.
  
    - Share In -The collection of downstream (sent to the client) share quality data.
    - Share Out - The collection of upstream (sent from the client) shares quality data.

The parameters are analyzed and graphically presented for each of these data points. These parameters are:

- Packet Loss
- Latency and
- Jitter.

JavaScript is required for chart generation. The code snippet for the visualization uses Golang’s templating system for code injection before being sent to the client’s browser.

```javascript

<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">
       // the JSON data(analytics) is provided when the HTML is generated from this template.
       var analytics = {{ .Data }};
 
	//  .StartTime & .EndTime is also provided when the HTML is generated from this template.
       const startTime = new Date('{{ .StartTime }}');
       const endTime = new Date('{{ .EndTime }}');
       var timeDiff = (endTime - startTime) / 1000;
 
// calls to the google charts API loads the chart presets we need.
       google.charts.load('current', { 'packages': ['line'] });
       google.charts.setOnLoadCallback(drawGraph);
 
       function drawGraph() {
		// ...
      		// details are redacted for brevity.
 }
// ...
      // details are redacted for brevity.
</script>


```

### Graphical Representation

The type of chart chosen for representation is a line chart. Line charts easily allow for comparing changes within a time interval for more than one data set, saving space. 

Google offers a public API for charts which is free and straightforward to utilize. This API is used to generate charts for visualization of video, audio, and sharing qualities from a Webex meeting. 



### Summary

Using Integrations with Webex's API, data for Webex meeting quality is visualized through REST API. The meeting room experience is represented creatively through line charts. The execution of the webserver has been successful. 



