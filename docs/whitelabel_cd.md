# Continuous Deployment for a Whitelabeled Application

At GOGov, we provide cities and municipalities with the ability to coordinate projects and improvements with their constituents.  Each customer gets an Android and an iOS app, and as we look to accelerate our growth, this process must be well documented and scalable.  

<p align="center">
  <img src="https://www.websitemagazine.com/images/blog/whitelabel-can.png">
</p>

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

## Getting Builds off Developer Machines

Our engineering team identified this as the number one goal, given the vast surface area for failure if done manually.  We knew we could use fastlane, and we created some modifications and supplements. 

We started by defining a build manifest.  Looking at the workflow above, we decided that when we internally approve customer changes, we'd generate an S3 object in AWS.  This would allow us to trigger a Lambda function that would create a commit in our mobile Flutter repository.  This commit would then invoke Github Actions workflows that would run tests and start the build process.

We felt that in the early stages of this process, buying a Mac Mini would be the easiest approach to running these workflows.  Each build takes about 15-20 minutes, and we'd burn through Github credits very quickly.  Setting up the Github Runner was relatively straightforward, but we ran into some [major problems we were able to work through](/docs/build_machine.md)

With the machine set up, it was time to start filling in the workflow to take a load off the manual labor.  

## Automating all the things!

We had to make a few modifications to some of the [Firebase Fastlane plugins](https://github.com/GoGovApps/fastlane-firebase-plugin), and we were able to generate FCM projects, add iOS clients, and download the GoogleServices.plist configuration file for build.

Using `wget` were able to download the graphical assets from the build manifest.  From here, we could generate all the [icon sizes](https://github.com/fastlane-community/fastlane-plugin-appicon)

## Fastlane Sessions

Fastlane sessions last about 30 days.  After that, you'll need to run through a login process that requires 2FA involvement.  Oh no! Our automation is ruined...or is it?

At GOGov, we never give into an engineering challenge.  We developed a cronjob that will run in our cluster and complete an entire 2FA flow, without user interaction.  The configuration based script is capable of updating k8s secrets and Github Action secrets.  We used RingCentral, which is used across our organization, to ingest an SMS message.  The script monitors an inbox after triggering the 2FA with [Fastlane Spaceship](https://docs.fastlane.tools/best-practices/continuous-integration/#authentication-with-apple-services).  Look ma! no hands!

## iOS push certificates

Things that expire are no fun to juggle manually.  As engineering teams, we have to deal with SSL certificates, rotating secrets, and one or two other things :) in our every day workflow.  Having a growing number of customers with scattered expiring push certificates is a nightmare, so we'll need to automate that too!

We solved this with Fastlane as well.  Each customer gets a branch named in coordination with the Bundle ID of the application.  This runs nightly, checks if a certificate is 45 days away from expiring, and generates a new one if necessary.  This new certificate can then be sent to FCM, using similar tooling as that in the build process.
