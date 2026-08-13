# The System Design Diaries: Building We Connect from Scratch

## Day 4 & Day 5 – Building Our First APIs and Welcoming the First Users

> *"A good architecture means nothing until users can actually use it."*

After spending the last few days planning our product and designing the architecture, today finally felt like the day every developer waits for.

It was time to write code.

As soon as I opened my laptop, our CTO reminded us of something important.

> **"Don't try to build everything at once. Build one feature completely, test it, and then move to the next."**

That became our development strategy.

---

# Building the First APIs

We started with the foundation of every application—authentication.

Without users, there is no social network.

We created our first REST APIs.

### Authentication

* Register User
* Login
* Refresh Token (planned for later)

### User

* View Profile
* Update Profile

### Posts

* Create Post
* View Post
* Delete Post

The excitement in the team was unbelievable.

For the first time, our application wasn't just a diagram on a whiteboard.

It was becoming a real product.

---

# Designing APIs Before Writing Business Logic

One habit our CTO insisted on was designing the API contract before writing any implementation.

Instead of immediately coding services, we first discussed questions like:

* What should the request look like?
* What should the response contain?
* Which HTTP status codes should we return?
* How should errors be handled?

This helped everyone understand the system before writing code.

A well-designed API becomes a contract between frontend and backend teams.

Changing that contract later becomes much harder.

---

# Keeping Module Boundaries Clean

Although everything still runs as one application, we continued following our modular approach.

The **Post Module** owns everything related to posts.

The **User Module** owns user information.

The **Like Module** handles likes.

The **Comment Module** manages comments.

One simple rule guided us:

> **A module should never directly manipulate another module's internal logic.**

Instead, modules communicate through well-defined interfaces.

This keeps the application maintainable as it grows.

---

# Our First End-to-End Flow

By the end of the day, we completed our first user journey.

1. Register
2. Login
3. Create a profile
4. Upload a post
5. View the post

Seeing that flow work for the first time was incredibly satisfying.

There were only a handful of test users.

But for us, it felt like millions.

---

# Day 5 – Our First Internal Launch

Today wasn't about writing new features.

It was about using our own product.

Every developer created an account.

Everyone uploaded posts.

We liked each other's content.

We commented.

Within a few hours, we started discovering bugs that no unit test had revealed.

* Duplicate likes
* Empty comments
* Missing profile images
* Slow feed loading after multiple posts

Instead of feeling frustrated, the CTO smiled.

> **"Congratulations. We finally have real problems."**

That sentence made us laugh.

But it also taught us something important.

Real software improves through usage.

Users always find scenarios developers never imagined.

---

# Measuring Before Optimizing

One developer suggested introducing Redis immediately because the feed felt slow.

The CTO stopped us.

> **"Before optimizing anything, measure it."**

We began collecting basic metrics.

* API response time
* Database query time
* Error rate
* Request count

At our current scale, everything was still performing well.

The feed felt slightly slower, but not because we needed Redis.

The problem was an inefficient database query.

Adding Redis would have hidden the symptom instead of solving the cause.

That was another valuable lesson.

> **Never optimize blindly. Measure first.**

---

# Looking Ahead

By the end of Day 5, We Connect was officially usable.

We had:

* User authentication
* User profiles
* Post creation
* Likes
* Comments
* A basic home feed

Only a few developers were using it.

But every great platform starts with a small group of users.

As we prepared for a wider internal release, one question remained:

> **What happens when hundreds... then thousands... of users start using We Connect at the same time?**

That question will lead us into the next chapter of our journey.

We'll discover why a single application server eventually becomes a bottleneck and why **Load Balancers** are one of the first scaling techniques every backend engineer should understand.

---

## Key Takeaways

* Build one complete feature at a time.
* Design API contracts before implementation.
* Keep module boundaries clean, even in a monolith.
* Test the application by actually using it.
* Measure performance before introducing new technologies.
* Solve the real bottleneck—not the imagined one.

> **"Simple systems become scalable because engineers solve real problems at the right time—not because they add every technology on Day One."**
