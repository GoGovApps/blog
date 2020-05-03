# Continuous Deployment for a Whitelabeled Application

At GOGov, we provide cities and municipalities with the ability to coordinate projects and improvements with their constituents.  Each customer gets an Android and an iOS app, and as we look to accelerate our growth, this process must be well documented and scalable.  

iOS and Android apps require a non-trivial number of tasks to get to a published state on their respective stores.  Let's take a look at the iOS workflow.

1. Customer provides basic branding information such as a background image, and colors.
1. Customer provides required information for the App Store.  This includes an Icon image, keywords, App Store name, description, etc.
1. Customer must now also create a Developer Account with Apple
1. Customer can now commit to their graphics, and let us know they are ready to deploy their app.

At this point, we review the customer's content and approve it.  From there, it's ready for build.  What happens next?

1. Register Bundle ID
1. Generate deployment certificate
1. Create Firebase project for Push notifications and Crashlytics
1. Generate push certificates
1. Upload the push certificates to the Firebase project
1. Store the FCM Server Key
1. Add the Bundle ID to the list of allowed Bundle Ids for Google Maps
1. Update the XCode project with the appropriate build variables based on the Customer inputs above
1. Update the XCode project with the correct Google Maps API key
1. Configure Provisioning Profiles for the project
1. Generate App Icon for all sizes required by the store
1. Compute a Build Number (remember, we are whitelabeling, so all customers will be different.  For example, if they want to update their Icon, they'll have to retrigger a build process)
1. Generate a signed build
1. Generate screenshots
1. Upload build to Testflight and iTunes Connect

Not much to think about, right?  Imagine having to go through this process for every single app.  If this is happening on developer machines, it's incredibly error prone.

This calls for some automation.  Here's some more context of our project

- Flutter
- Firebase Push
- Firebase Analytics
- Firebase Crashlytics
- Google Maps

Many of the steps above are automatable with the vast toolset provided by Fastlane.  But the fact we are whitelabeling applications adds a layer of complexity for just about every step.  
