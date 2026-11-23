LAB: Storing and Serving Delivery Photos for Onward Logistics 

Module: Databases & Storage on AWS  |  Week (M) 5 

1. Business Scenario 

Onward Logistics is launching a new service for delivery photos. Riders photograph each delivery as proof, and customers and merchants view those photos later in a gallery. The photos are evidence in disputes, and both the number of photos and the number of people viewing them grow every month. The service is a single containerized full stack application, and each photo comes with details such as a title and a description. 

In a planning meeting, four leaders set out what they need: 

Chief Technology Officer: “Customers in different cities should see their delivery photos quickly, however many people are looking.” 

Head of Security: “Merchants and customers own these photos. They must not be viewable from outside our system: no direct access to where they are stored, and no way to pass them around. We cannot afford copyright complaints or misuse.” 

Head of Operations: “Delivery photos are our proof in disputes. Rebuilding the application is easy, and losing a photo is not. If the whole Region we run in, or the place the photos are stored, is lost, we must still have every photo.” 

Finance Lead: “Photos pile up and viewing grows with them. Show me what storing, protecting and showing the photos costs, and why you chose it.” 

For background on the company, its stakeholders and its standards, read the Client Profile: Onward Logistics. 

2. What the Client Expects 

These are the objectives. How you achieve them is your design. 

Store the delivery photos in the AWS storage service best suited to them and justify the options you rejected. 

Keep each photo's details (title, description and other details) in a relational database and link every record to the photo it describes. 

Keep photos viewable only through the Onward system and never directly from where they are stored, keep the database unreachable from the internet, and keep credentials out of code. 

Keep photos and their details recoverable after an accidental deletion or a failure. 

Keep every photo even if an entire Region, or the place the photos are stored, is lost. 

Show the Finance Lead what the service costs and justify your choices on cost and resilience. 

Prove to the CTO, the Head of Security, the Head of Operations and the Finance Lead that every objective is met. 

Stretch objective: beyond making photos viewable, make them load quickly for customers in different locations while keeping the cost of viewing down as it grows. This needs research beyond what was taught in class: find what AWS offers for this, compare the eligible options, and justify the settings you choose. 

3. Mandatory Technical Requirements 

Provision everything with AWS CloudFormation. Nothing may be created manually in the console. 

Deploy the infrastructure using CloudFormation GitSync from a version controlled repository. 

Ensure the infrastructure is configured to be highly available and strictly matches the requirements of the Head of Security. 

Build the application in the language your instructor assigns (Python or Java), as a single full stack application. It must let a user upload a photo with a title, description and other details, show a gallery of the uploaded photos with their details, and display a page showing your full name and the lab name. Sign in is not required. 

Run the application on Amazon ECS with Fargate, with tasks in private subnets behind an internet facing Application Load Balancer. 

Store the photo details in a relational database managed by Amazon RDS. The database must not be self managed. 

Only the storage of the photos needs protection against the loss of a Region. The application and the database stay in a single Region. 

Keep application code and infrastructure code in separate repositories. 

Follow the company standards in the Client Profile, including naming and tags. 

Do not use inline policies anywhere in the infrastructure templates. 

4. Stage 1: Discovery (Before You Build) 

Complete a discovery report before you build anything, using the Discovery Report Template. Fill it in and share the link when you submit. It has two parts: 

What you found: common solutions that exist for this kind of problem, and what happens with no solution. 

Business value: your chosen approach, the AWS services you plan to use and how each will be configured, and who benefits. 

Your report is your written design. For every stakeholder requirement above, show how your design meets it, the key technical settings involved, the options you considered and rejected, and the research that supports your choice. 

5. Stage 2: Build and Prove 

Design and build a solution that meets the objectives and the mandatory technical requirements. You decide the design, including the storage and delivery choices, the database design, the security rules, and how the photos are protected. Create an architecture diagram of your solution. Be ready to demonstrate the client's checks live. 

The client will check that: 

A photo uploaded through the application appears in the gallery with its title, description and other details. 

The photo file is not stored in the database, and the database cannot be reached from the internet. 

A direct request to a photo's storage location is refused, and photos load only through the Onward system. 

No credentials appear in the repositories or the application code. 

A deleted photo or record can be recovered. 

A photo uploaded today also exists in a second Region, and you can explain what is and is not protected if the primary Region is lost. 

The application keeps working if a container or an Availability Zone is lost. 

Stretch: photos load quickly for customers in different locations, and you can explain the cost of viewing. 

You can explain what your design costs and which alternatives you rejected. 

The whole setup can be rebuilt from the repositories. 

6. Deliverables 

Discovery report: a link to your completed report (what you found and business value). 

Lab deliverables: links to the repository holding your CloudFormation infrastructure and the repository holding your application code, Dockerfile and related build files. 

Architecture diagram: a link to a diagram showing the resources deployed in both Regions, which are public and which are private, and how a photo flows from upload to viewing. 

Live review: walk through your code, demonstrate the client's checks, and defend your reasoning. 

7. Rubric 

Discovery report (what you found and business value): 20% 

Core solution meets the objectives and technical requirements: 40% 

Stretch objective (fast, low cost viewing): 10% 

Architecture diagram: 10% 

Demonstrate understanding of solution in live review: 20% 

8. Submission 

Submission Type: Form link (discovery report link, repository links and architecture diagram link) 

Grading: Live Review Session 