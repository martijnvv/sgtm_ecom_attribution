# sgtm e-commere attribution

Based on (https://github.com/gtm-templates-knowit-experience/sgtm-ga4-ecom-item-list-promo-attribution)[GA4 - Item List & Promotion Attribution - SGTM Variable (Server)]

# Steps to configure

## Google Cloud, Firestore & Cloud Functions Setup
If you want an easier understanding of cost, it’s recommended to create a new Google Cloud Project for the Firestore setup.

### Firestore Setup
* Select a Cloud Firestore mode
  * Select Native Mode
* Choose where to store your data
  * Create Database
  
If you are running Firestore in a different Google Cloud Project than Server-side GTM, you must add the SGTM service account to the Firestore project via IAM.

Grant the service account a Cloud Datastore User role to give SGTM access to the Firestore project.

* If Server-side GTM is running on App Engine, add the Server-side GTM App Engine default service account to the Firestore project.
* If Server-side GTM is running on Cloud Run, add the Server-side GTM Compute Engine default service account to the Firestore project.

### Delete outdated documents in Firestore
Use time-to-live (TTL) policies to automatically delete outdated documents.
In Firestore, go to Time to live (TTL).

* Click Create Policy
* Collection group: ecommerce
* Timestamp field: expire_at
* Click Create button
  
### Cloud Functions
To be able to use TTL, the TTL field must be of type Date and time. At the time of writing, SGTM can't store data in this format to Firestore. To get around this we use Cloud Functions to write Date and time to Firestore. Note: This increases Firestore reads & writes.

#### Create function
We need to create 2 functions; create & update. These functions will listen to changes in Firestore, and will take a Timestamp set by SGTM in a unix format (number), and rewrite that number to Date and time.

Configuration
* Basics
  * Environment: 1st gen
  * Function name: ga4-int_attribution-date-time_create
  * Region: choose a region close to or the same as Firestore
  * Trigger type: Cloud Firestore
  * Event type: create
  * Document path: ecommerce/{docId}
* Runtime
  * Memory allocated: 256 MB (128 MB may also work)
  * Other settings as is
* Connections
  * Allow internal traffic only

Code
* Runtime: Node.js (latest version)
* Source code: Inline Editor
* Entry point: makeDateTime
  
index.js

```
const Firestore = require('@google-cloud/firestore');
const firestore = new Firestore({
  projectId: process.env.GOOGLE_CLOUD_PROJECT
});

exports.makeDateTime = event => {
  const curValue = event.value.fields.expire_at.doubleValue;
  if (curValue && typeof curValue === 'number') {
    const affectedDoc = firestore.doc(event.value.name.split('/documents/')[1]);

    let newValue = new Date(curValue);
    newValue = new Date(newValue.setDate(newValue.getDate() + 7)); // Set TTL to 7 days from now.

    return affectedDoc.update({
      expire_at: newValue
    });
  }
};
```
The reason for setting TTL to 7 days from now is to reduce TTL deletes. If we set TTL today, and the user comes back in a couple of days, TTL deletes will be done twice for this user.

```
package.json
{
  "name": "sample-firestore",
  "version": "0.0.1",
  "dependencies":{
   "firebase-admin": "11.3.0",
   "firebase-functions": "4.1.0"
}
}
```

### Deploy function.

* Now create a identical function, but select Trigger type update instead.
* Name this function ga4-int_attribution-date-time_update
  
Cloud Functions setup is now completed.
