Title: Analysis
Date: 2018-12-31 10:00:00
Category: Start Here
Tags: problems

# Current Code base #

## Abstract ##

I do not believe the current application platform to be a good choice
to use going forward.  There are general problems with the transaction
handling and the code problems will make fixing those very difficult.I
am not recommending a rewrite, since history has demonstrated that
rewrites are almost always the 
[Wrong Choice](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/).

## Back-end ##

The current Django back-end is very fragile.  Fragility in this case isn't
a metric of code quality,  and the code itself isn't broken per se.
Syntax inspection shows 14 warnings and 735 weak warnings, which isn't
bad.  Nearly all of the weak warnings are PEP8 related and the 14
warnings are mostly related to a casualness of variable naming
and access (these kinds of problems are also common).  The fragility
here is that the code is very hard to modify and it is spread out in
such a way that small changes require much larger Line Of Code (LOC)
changes than they should.  This when combined with low test coverage
means refactoring is much more difficult and error prone than it
should be and the code base cannot be refactored[^1] with confidence.

The back-end is broken into many Django applications, though each of
these is not really a discrete part (with the possible exception of
surveys).  There are 9 applications and a total of 15 models.  I'm
going to focus on the models here because they are the foundational
components of any Django application.

The models are all tightly coupled and probably should be moved into
one or two Django apps.  They are also very large, on average, and
should be decomposed into smaller models. For example,  the donations
app `Transaction` model has 16 fields,  several of which are specific
to M-PESA transactions and should be extracted into a related model.
However,  because the serializers (which are required to support the
web front end) are built in a way that is tightly coupled to this
model,  this change would require a lot of work and would likely need
to be coordinated with changes to the front-end code base. Changing one
application is difficult, changing two applications in lockstep is
unwise.

### Database ###

Postgres is a good choice for any performant web application.
However,  running a Dockerized Postgres makes development setup
cumbersome.  There are no seeds for db setup and creating a reliable
development setup requires help from at least one other developer.

Also, it seems that Postgres was chosen mostly to support the JSON
datatype in Django.  This is only used in one field in one model. This
could be converted to using json stored as a string which would allow
SQLite for local development and make developer setup much easier.
It's also not apparent that this one JSON model is appropriate since
the data from this field is only used in one other location in the
application. Further complicating this is the `Transaction` model
which also has a json data field but is NOT a JSON datatype.

The last issue with the database is that transaction information is
stored in the DB without other mechanisms for verification. There are
no triggers or stored procedures to allow for automated double entry
nor any mechanism to detect tampering or data loss.  Much of the
information about transactions should probably be stored in a logged
object-store with well known archiving and tracking.  Without that
data, audits of transactions would be much more difficult and time
consuming.

## Front end ##

I don't think a Single Page Application is warranted for this
application.  It would be much easier and less code to implement the
UI in Django.  The existing API is tightly coupled to the UI, which
makes it far less valuable to have.  It would be better to expose a
data API for partners to use at a point in the future when there are
enough partners to communicate with about their needs.  There is a
jargon work in tech YAGNI:   You Aint Gonna Need It and this
definitely applies here.  You don't need an API until you have enough
users to justify the cost of building one.  For now,  this API is a
YAGNI item.

## Transaction Handling ##

Generally speaking, the transactions are not stored in a secure,
auditable manner.  The transaction and payment API also has no
mechanism to alert anyone if transactions fail.  For example, if the
M-PESA account does not have sufficient funds and 5 donations come in
those payments will fail.  Fixing that problem is manual and there
doesn't seem to be any mechanism for dealing with payment failures at
any point.  While charge-backs are very difficult and would not apply
here, other failures are much more common.  The system also does not
keep a durable, auditable record of payments (which was discussed
earlier) nor does it have a way to stop, restart, and retry payment
activity.

## Conclusions ##

The system could probably be launched as it is now. Most of the issues
with the code are future problems.  The weaknesses in the transaction
handling would give me pause, however.  Those weaknesses could be a
major problem if anything went (even slightly) wrong. Fixing those
weaknesses will require grappling with the future problems.


# Proposed Solutions #

## Abstract ##

While I am not recommending a rewrite, I am recommending a totally
different approach to the early phase of this project.  It is too
early to write a system to manage this process and so it would be
better to use existing tools to accomplish the same goals.  There are
well-known e-commerce and cloud solutions that can handle everything
except automated M-PESA payouts.

## Collecting Partner/Recipient Information ##

Partners need to be able to enter the required information about their
participants.  This information can easily be captured using Google
Forms.  These forms have the ability to upload images, enter text,
etc. and even support conditional logic to make your form as flexible
and you want.  These forms can be modified by non-technical users and
are reasonably secure.

Each participant organization would be supplied with their own form,
which would store the submitted information in a spreadsheet hosted by
Google Docs.  Auditors can be permissioned to view, and optionally
edit, these spreadsheets and changes can be tracked.

On-boarding new partner organizations would require the following
steps:

  1. Create a new copy of the form
  2. Email the form submission link to the partner
  3. Share the backing sheet with the auditors.
  
Here is an
  [Example Form](https://goo.gl/forms/210JM9Hz7h3Spwx83).


## Collecting Donations ##

PayPal and Stripe both support simple payment buttons.  You can see a
live example [here]({filename}/buttons.md).  Many buttons can be
created and, since they are provided as HTML fragments, they can be
embedded in any web page. These pages can be served from ads (Facebook
ads have excellent support for action buttons like this).  

It would be a manual step to create buttons from participant/partner
information.  However, this is a simple process and each button can
support several "programs" which can be used to select specific
participants.  

Site management services like SquareSpace or even WordPress can be
used to create pages from the information collected from partners and
buttons can be attached to those pages.  PayPal supports M-PESA
transfers directly, unlike Stripe.

## Making Payments to Recipients ##

### Manual Payments ###

Safaricom allows M-PESA payments to be made via their website, though
it does require a Kenyan mobile number.

PayPal supports transferring balances from PayPal to M-PESA directly
using [PayPal MobileMoney M-PESA
service](https://www.paypal-mobilemoney.com/m-pesa).  This service can
be used to transfer the money collected via donations directly to an
M-PESA account for disbursal.

Manual payments would require more auditing scrutiny and are somewhat
labor intensive.

### Automated Payments ###

This is the only part that might require some coding.  Payment
providers have the ability to use web-hooks which could be used to
automatically transfer funds and make payments via any mechanism that
provides an API.  These web-hooks can be implemented using AWS Lambda or
GCP Cloud Functions and are cost effective and easily auditable.  I
would recommend running manual payments for a short period to
establish the kinds of payment volumes you might have and to work out
any kinks in the payment workflow.  Once the manual workflow is
established and understood, it can be automated. This is a much more
robust way to build automated services.

## Conclusions ##

Not a rewrite but a change in direction.  Creating a simple site with
a few payment buttons and a Google Form would be much easier than
fixing the code. This process would also allow the product to be
launched in a limited fashion and a shakedown of the whole process to
be performed.  It's less risky and much easier to change direction
with this strategy.


[^1]: Refactoring is a process by code is changed without altering the
    functionality or larger parts of the application. https://en.wikipedia.org/wiki/Code_refactoring
