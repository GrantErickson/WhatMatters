This is a repo that will have a markdown file that is a hierarchical list of all components of a website, specifically one built in .NET. It should include the following top-level ideas. Please complete the list based on the sample level of detail.
The idea is that each end item should be given a level of scrutiny that a developer should apply when reviewing AI code. For example, security and core data model things need lots of review. Database migrations with EF should be looked at but less so. All security code should be looked at carefully. Front end security code should be looked at. But other CSS code should be verified that it isn't sloppy and follows conventions. 
This app assumes that the website and the API are separate and that there are libraries for the data project. It is also assumed that it will be deployed to Azure and will use features like Key Value, Blob Storage, AI, and etc. It also assumes that Vue or React is used as a front end framework with a library like Vuetify or Tailwind. 
Add and modify items from this list to make it complete. It should be complete as to show the level of detail needed for identifying where to make sure developers are reviewing code, but not be overly detailed. 
Also push back on things that can be put into instruction files that developers can start ignoring. What best practices fall into this category?

1. API
1.1 ASP.NET Pipeline
1.2 Security Model
1.3 Endpoints
1.3.1 Endpoint Security attributes
1.3.2 Endpoint data being passed
1.3.3 DTO mapping to prevent data exposure
1.4 Nuget Packages

2. Web Site
2.1 Pages
2.1.1Marketing Site
2.1.2 Login
2.1.3 Main Page
2.1.3.1 Usage of framework
2.1.4 Other pages
2.2 NPM packages selected with versions
2.3 Component architecture
2.4 Client-side models

3 Data project (containing EF classes and business logic)
3.1 Class design
3.2 Migrations
3.3 Business Logic and Services
3.4 Security aspects

4. Deployments
4.1 YAML
4.2 Terraform

5. General architectural concerns
