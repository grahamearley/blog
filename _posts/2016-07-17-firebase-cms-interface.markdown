---
layout: post
title:  "Using Firebase as a Parse-like CMS with Google Sheets"
date:   2016-07-17 13:37:10 -0600
---

In the months following [Parse's shutdown announcement](http://blog.parse.com/announcements/moving-on/), I was in school and was putting off the migration of my current Android app that utilized Parse. I didn't want to have to host my own Parse server, but it was looking like that was my only option. However, in May, the solution arrived: Google's Firebase service underwent a [major upgrade](http://firebase.googleblog.com/2016/05/firebase-expands-to-become-unified-app-platform.html). 

Before even beginning with Parse, I had considered using Firebase as my backend-as-a-service provider. The problem was that my app requires file storage, which Firebase didn't offer and Parse did. However, the new Firebase *does* have file storage, which allows me to easily download server-side data in my app.

Although server integration is essential to my app, the integration itself is not particularly complex or messy. That is, it wasn't a problem to switch all the models and adapters to use Firebase instead of Parse. After a day of refactoring, I was completely out of Parse's ecosystem and into Firebase's.

Now, the key part about my app's database is that the data is maintained by my client (an education group), not by me. The app is a platform for lesson plans, and my client manages the lesson files and information that gets uploaded to the server. As the content managers are not programmers, the familiar, spreadsheet-style Parse Dashboard was very useful for us. It was very intuitive to add new rows to the sheet to add a new lesson.

<!--more-->

![The Parse Dashboard](/blog/img/parse_dashboard.jpg)

However, the interface for the Firebase database is much closer to writing pure JSON. Each entry has a parent-child hierarchy, and for each new entry, you must spell out both the key and the value. An entry in the database looks like this:

![The Firebase JSON interface](/blog/img/firebase_json.png)

When someone else is maintaining the data, this parent-child format is more worrisome than the spreadsheet format. In the spreadsheet format, the developer can control the names of all the columns, (e.g. `lessonName` or `subject`) and be confident that queries to these columns' name in code will not come up empty.  However, with Firebase's JSON format, there's more room for error. A simple mistake while typing in the key of the entry could lead to a `NullPointerException`. For example, if you are adding 500 lessons to the database by hand, it's not hard to imagine that on lesson 392, you might type `subejct` instead of `subject` for the Subject key. Now we've got an improperly formatted lesson in the app.

To make this easier for the client and to give me more confidence in the results my queries were returning, I set out to get that spreadsheet format back. To do this, I set up a Google Sheet that interfaces with Firebase using a Google Apps Script. Each time the spreadsheet is updated, its rows and columns get converted into a JSON object that is sent to Firebase.

From a Google Sheet, you can add a script that exports the data to a JSON format by opening the *Script Editor* in the *Tools* menu. In the script editor, open the *Resources* menu and click on *Libraries* to add the Firebase library. In the "Find a Library" bar, enter the key `MYeP8ZEEt1ylVDxS7uyg9plDOcoke7-2l` to find the Firebase library. Choose the latest stable version.

The function that converts the sheet to JSON will depend on the structure of your data, but there's a simliar pattern to follow. First, you need to get an instance of your database. You can find the URL of your database at the top of the database page in Firebase, and you can find your database secret (or generate a new one) in the *Database* tab of Firebase's *Project Settings*. Then, write a function in your Google Apps Script that looks like this:

{% highlight javascript %}
function writeDataToFirebase() {
 var firebaseUrl = "https://_________.firebaseio.com/";
 var secret = "Your database secret from Project Settings > Database";
 var base = FirebaseApp.getDatabaseByUrl(firebaseUrl, secret);

 // … More to come!
{% endhighlight %}

Now, in this function, you need to read the spreadsheet, which you obtain using the ID at the end of the spreadsheet's URL (the big string of numbers and letters). Once you have the whole spreadsheet, you will also want to grab the specific sheet of the spreadsheet where your data is, and the data range of that sheet. In the following code, we're getting the first (0th) sheet in the spreadsheet.

{% highlight javascript %}
// Access your spreadsheet and its data:
var YourSpreadsheet = SpreadsheetApp.openById("ID from your sheet's URL");
var dataSheet = YourSpreadsheet.getSheets()[0]; // first sheet
var data = dataSheet().getValues();
{% endhighlight %}

Now, iterate through the sheet's rows and store each column to a specific variable in the JSON object we're outputting.

{% highlight javascript %}
// Create new JSON object to import to Firebase:
var dataToImport = {};
for(var i = 1; i < data.length; i++) {
   dataToImport[gradeNumber] = {
      lessonName: data[i][0],
      gradeLevel: data[i][1],
      subject: data[i][2]
  };
}
{% endhighlight %}

Finally, you need to add your new JSON object to your Firebase database.  We'll add it under the name `Lesson`. If an object already exists under `Lesson`, this will overwrite that object. However, since we'll only be maintaining this object from the Google Sheet, we're not worried about any data loss — any overwrites will simply be keeping Firebase up to date with the spreadsheet. To write the `dataToImport` JSON object to Firebase under the key `Lesson`, we'll use the `base` variable we created earlier, like this:

{% highlight javascript %}
base.setData("Lesson", dataToImport);
{% endhighlight %}

Combining all this gives us this function:

{% highlight javascript %}
function writeDataToFirebase() {
 var firebaseUrl = "https://_________.firebaseio.com/";
 var secret = "Your database secret from Project Settings > Database";
 var base = FirebaseApp.getDatabaseByUrl(firebaseUrl, secret);

 // Access your spreadsheet and its data:
 var YourSpreadsheet = SpreadsheetApp.openById("ID from your sheet's URL");
 var dataSheet = YourSpreadsheet.getSheets()[1];
 var data = dataSheet().getValues();

 // Create new JSON object to import to Firebase:
 var dataToImport = {};
 for(var i = 1; i < data.length; i++) {
    dataToImport[gradeNumber] = {
       lessonName: data[i][0],
       gradeLevel: data[i][1],
       subject: data[i][2]
    };
 }
 
 // Add data to Firebase:
 base.setData("Lesson", dataToImport);
}
{% endhighlight %}

The final touch is to make our function get called each time the spreadsheet is updated. You can do this by clicking the *Resources* menu in the Google Apps Script editor and selecting *Current project's triggers*. Use this tool to add triggers that will call your function based on certain events. You can have the function get called whenever the spreadsheet is edited (or when other key events happen to the spreadsheet, like changes or opens) or you can have the function get called at certain time intervals. 

With this new, more familiar interface set up, the database is ready to face the clients as a content management system with little risk of invalid data formatting (use Google Sheets's data validation features to reduce that risk even further).


✌ 

Questions? [Email me.](mailto:graham@grahamearley.website)