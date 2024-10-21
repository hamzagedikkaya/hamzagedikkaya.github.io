---
layout: post
title: Leave Master Project
author: Hamza Gedikkaya
categories: 
  - Rails Projects
excerpt_image: /assets/images/posts/first_gif.gif
banner:
  image: /assets/images/posts/first_post.jpg
  height: "50vh"
tags: 
  - Leave Master Project
  - Gem
top: 1
---

### Leave Master Project: Step-by-Step Development Process

This project focuses on developing a **Leave Management System** where users can efficiently track and manage their annual leave. The system will support essential features such as requesting, approving, and monitoring employee leave. Managers will have access to comprehensive leave reports, ensuring smooth operations without disruptions.

## Project Overview

In the **Leave Master** application, employees can submit leave requests for annual vacations, sick leaves, and other time off categories. The system helps managers monitor and manage these requests, ensuring that no department is left understaffed while allowing employees to take their entitled leave.

Key features of the Leave Master project include:
- **User Authentication**: Employees and managers can log in and access their personalized dashboards.
- **Leave Request Submission**: Employees can request different types of leaves, specify start and end dates, and provide comments.
- **Approval Workflow**: Managers can approve or reject leave requests with real-time notifications to employees.
- **Leave Balances**: Employees can track their remaining leave days for the year.
- **Admin Reports**: HR and management can generate reports on leave patterns and ensure compliance with leave policies.

## Technologies Used

We are leveraging the following modern technologies to build this project:
- **Ruby**: 3.3.0
- **Rails**: 7.2.1.1
- **PostgreSQL**: A robust and scalable relational database to handle leave records.
- **JavaScript (ESBuild)**: Efficient bundling and building for modern JavaScript development.
- **Bootstrap or Tailwind**: Choose your preferred CSS framework for styling.

---

## Project Initialization

To kickstart the project with **Bootstrap** as the CSS framework, you can use the following command to generate your Rails project:

```bash
rails new leave_master --javascript=esbuild --database=postgresql --css=bootstrap
```

If you prefer using **Tailwind CSS** for a more utility-first styling approach, you can initialize the project with:

```bash
rails new leave_master --javascript=esbuild --database=postgresql
```

After the project is created, navigate to your project directory and set up the database by running the following commands:

```bash
# Navigate to your project folder
cd leave_master

# Set up the database
rails db:setup

# Apply the migrations to generate the schema.rb file
rails db:migrate
```

Once the database is set up and migrations are applied, you can make your initial commit to track the project setup with:

```bash
git add .
git commit -m "chore: initialize LeaveMaster project and setup database"
```

You can view the relevant commit [here](https://github.com/hamzagedikkaya/leave_master/commit/ad686a55ca41027e38b2032d1aa2206ae4121ca0).

---

## Tailwind CSS Setup

To set up Tailwind CSS in your Rails project, follow these steps:

```bash
./bin/bundle add tailwindcss-rails
./bin/rails tailwindcss:install
RAILS_ENV=development bin/rails assets:precompile
git commit -m "chore: add Tailwind CSS setup"
```

These commands will install Tailwind CSS, set up the configuration, and precompile your assets. If your local CSS changes under the `assets` directory are not reflected, you might need to re-run the precompile command to apply those changes.


## Home Page

To create the home page, we first generate a controller by running the following command:

```bash
rails generate controller main home
```

Next, we update the routes file by changing the line: `get "main/home"` to `root "main#home"`.

Now, we can start the Rails server with:

```bash
rails server
```

When we navigate to localhost:3000, we will be automatically redirected to the home page.

Finally, commit your changes with:

```bash
git add .
git commit -m "feat: generate main controller and update routes for home action"
```

You can view the relevant commit [here](https://github.com/hamzagedikkaya/leave_master/commit/41901c72b8d65712bcf5be94884198e77f916ff3).
