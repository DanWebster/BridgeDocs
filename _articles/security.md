---
title: Security Overview
layout: article
---

<div id="toc"></div>

Sage has extensive internal policies which govern the deployment of Bridge Services to ensure compliance with legal and ethical obligations governing human health research.  We've undergone security reviews by numerous industry and acadmeic partners, and have our systems and processes regularly audited by external security specialists to ensure compliance with HIPAA and other regulations.

#Cloud providers used by Bridge Services
Sage relies on a number of external commercial services to provide the lower level infrastrucutre components for Bridge Server:

* **Amazon Web Services** provides the base layer for all Bridge Services. AWS is the world-wide leader in cloud computing and provides security measures at the physical and network layers sufficient to comply with strict federal requirements including [the standards of HIPPA](http://aws.amazon.com/compliance/). Coded study data are stored at rest in Bridge using a combination of [Amazon S3](https://aws.amazon.com/s3/) and [DynamoDB](https://aws.amazon.com/dynamodb/). A number of asynchronous background processes run on [Amazon EC2](https://aws.amazon.com/ec2/) workers or as [Amazon lambda](https://aws.amazon.com/lambda/) functions.

* **[Stormpath](https://stormpath.com/)**, a cloud-hosted authentication as a service provider, is used to manage user account information including Personally Identifying Information.  Using a third-party service specializing in user management and authentication allows us to achieve solid security around users’ personal account data and ensure a clean separation between personal account data and the de-identified study data that can be used for research. Stormpath is itself hosted by Amazon.  

* **[Heroku](https://www.heroku.com/)**, is used to host the server runtime and provides the ability to rapidly adjust server bandwidth to load. No data is stored in locally in Heroku servers; this tier communicates with the underlying storage layers in AWS and Stormpath, and with client applications via HTTPS.  Heroku is itself hosted by Amazon.

* **[Redis](https://redislabs.com/)** is used as a in-memory cacheing layer by some services.  Redis is itself hosed by Amazon. 


#Storage of Personally-Identifying User Account Data by Bridge Server
Bridge has been deployed in research studies following two main strategies:

1. **Study participants self-enroll in the study electronically.**  This strategy is used when studies allow participants across the country to self-enroll in the study electronically, as in many of the public Research Kit studies.  In this case, the app provides  minimal identifying information such as first name, last name, username, Date of Birth, and email address to the server.  This information is sufficient to create an electronic record of consent to participate in a mobile study. Bridge also provides tools to the research team to facilitate recontact of study participants when permitted by the study protocol.  This strategy has the broadest possible enrollment, and allows for study designs in which participants never physically interact with the study team.

2. **Consent and registration is managed by a 3rd party, and participants use Bridge Server anomously**  This strategy is often used when the study app is a complement to a traditional clinical study, in which participants are enrolled in person during a clinic visit.  In this case, a 3rd party research team may maintain all personally-identifying information about a study participant.  Either Sage, or the research team may assign anonymous identifiers which are used to identify participants by Bridge Server.  In this case no PII is stored in Bridge server, at the cost of prohibiting auto-enrollment in the study through a publicly-available app.  

In either strategy, Bridge stores account information in a back-end database distinct from the one that stores the study data.  Currently, the Bridge server uses a commercial authentication system provided by Stormpath to store and manage such user account data.   This way, in the rare event that either database is compromised, the connection between a person and her/his health data will remain protected. Most account data is stored as encrypted key value pairs in Stormpath, and the Bridge server encrypts / decrypts user data as it is stored / retrieved from Stormpath, using a key generated and controlled by Sage Bionetworks. The exception to this is user email which is managed securely by Stormpath, but not separately encrypted by the Bridge server.  This allows Stormpath to take on some aspects of user account management such as email validation and password resets.

Mobile or web clients do not communicate directly with Stormpath, but instead with the [Bridge API](/articles/rest.html) backed by the Bridge server running on Heroku.  In our current implementation, the Bridge server acts as a proxy to Stormpath for management of user account data.  Although we have identified no concerns with hosting user account information in Stormpath, our architecture is such that, if needed, we can move to a different service, or even to an in-house implementation. 

When the Bridge server makes calls to Stormpath, we supply an API key provided by Stormpath so they can authenticate call coming to our instance.  The Bridge server uses standard HTTPS to communicate with Stormpath, thus we use standard certificate authorities to authenticate Stormpath to ourselves.  At the time of this being written, the Stormpath certificate is issued by DigiCert High Assurance EV CA-1.  As we are using Stormpath's Java SDK library, the communication is handled by Java 7 runtime.  The DigiCert's certificate is included in the Java 7's trust store.  The connection uses TLS 1.2.  It is encrypted using AES_128_GCM and uses ECDHE_RSA as the key exchange mechanism.  

#Storage of Study Data
Study data, defined as survey responses and mobile sensor data, is stored in Bridge separately from user account information.  Stormpath generates and assigns it’s own unique identifier to each user for it’s own internal use, which Bridge uses as a publicly accessible account identifier.   In addition, for each user in a study, the Bridge service generates its own pair of universally unique identifiers (UUIDs).  The first UUID is called “Bridge account ID” and is generated by the Bridge Server on account creation, encrypted, and saved in Stormpath.  Encryption uses 256-bit AES, GCM-mode.  The AES cipher is created using [Apache Shiro](http://shiro.apache.org/).  The actual encryption is provided by the latest version of [Bouncy Castle](https://www.bouncycastle.org/).  Bridge servers are currently deployed as Heroku stacks.  As a result, the encryption key is set as an environment variable on the Heroku admin console to pass to the Bridge servers.  Only the Bridge technical team have access to the Heroku admin console.  Additionally, each stack (local, development, staging, or production) is assigned its own encryption key.  The second UUID, called “participant health code”, is used internally by Bridge as the key to user’s study data.  If the same user enrolls in multiple studies, multiple account IDs and health codes will be generated to keep data for each study isolated.  A one-way mapping is set up from account ID to health code.  Currently the mapping is implemented as a DynamoDB table with the account ID as the hash key.  The workflow is illustrated by the following diagram:

![Personal Health Data Encryption](/images/anonymization1.png)

Thus, Bridge never links the Stormpath account ID directly to the study data.  When an authenticated user makes a call to create, update, or delete his/her data, the Bridge server retrieves the encrypted account ID from the user profile, decrypts the key, and uses the key to find the user’s health code in the map.  The mapping may exist for a time in memory on the Bridge server, but is never stored in a way accessible to someone who gains access to the back-end systems. This design limits the view of the entire map, ensuring that even Sage Bionetworks engineers with access to back-end systems would find it exceedingly difficult to reidentify patients in this system.

Bridge stores encrypted study data in a combination of AWS DynamoDB and AWS Simple Storage Service (S3). The only unencrypted data stored in either system will be the study participant ID, the study ID, the ID of the data module (defines the data schema), and the time the data was captured, stored in Dynamo for each time point at which data is collected. This allows Bridge to support time-range queries for a user to retrieve his / her own data. This design allows clients to complete personal data histories for participants, even if data is collected from multiple devices / interfaces. Large binary study data such as voice recordings, and bundled uploads will be stored in S3, using Amazon’s server-side encryption. Amazon manages the server-side encryption transparently for us. It currently uses 256-AES-GCM.

![Sequence Diagram](/images/security2.png)

Process for transferring health data from client to Bridge server: 

1. Data is encrypted by the mobile client using Cryptographic Message Syntax (CMS). The CMS public key is generated by Sage using 2048-bit RSA and is supplied to the app developer. Each study is assigned a different public key. Data will be cached on the device for upload to the server pending network availability. 

2. A background process on the client OS calls the service to get a location to upload data to, returned as URL pointing to an AWS S3 bucket dedicated for file upload that is part of the Bridge stack. 

3. Upload of the encrypted data file over S3 is via HTTPS. 

4. A call to Bridge Server informs the server that the upload process is complete. 

5. An asynchronous server process decrypts and validates the file using a private key held only by Sage Bionetworks. 

6. Metadata is written to Dynamo to record the user, study, and time of data collection. Research data is written to a S3 bucket dedicated for each particular study. For more details see our policies around Key Management.

#Researcher Access to Study Data
Bridge provides no APIs to allow researchers to query the study data in real time.  Instead, authenticated researchers on the project team can trigger an export of the aggregated study data from all participants in their study in which participants are identified only by their unique study data ID.  This ensures that the researcher cannot link back any particular records to any particular participant.  
