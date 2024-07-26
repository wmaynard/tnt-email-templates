# Introduction

As part of the Rumble Account integrations, email support was added.  Emails are sent out for various events, and the platform email templates are maintained from this repository.  Once pushed, the changes can be rolled out on a per-environment basis using Gitlab's pipelines.

## Acknowledgment

These email templates were originally created for Rumble Entertainment (which later became R Studios), a mobile gaming company.  These templates were used only for our account access / in-house 2FA alternative to third-party SSO.

R Studios unfortunately closed its doors in July 2024.  This project has been released as open source with permission.

As of this writing, there may still be existing references to Rumble's resources, such as Confluence links, but their absence doesn't have any significant impact.  Some documentation will also be missing until it can be recreated here, since with the company closure any feature specs and explainer articles originally written for Confluence / Slack channels were lost.

While Rumble is shutting down, I'm grateful for the opportunities and human connections I had working there.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE.txt) file for details.


## Background

Amazon SES requires us to send them a JSON file containing all of our template information for an email:

```
{
    "Template":
    {
        "TemplateName": "__name__",    // Alphanumeric characters and dashes only.
        "SubjectPart": "__subject__",  // Subject line of the email.
        "HtmlPart": "__html__",        // The HTML content for enabled clients.
        "TextPart": "__text__"         // The text content for clients with HTML disabled.
    }
}
```

When pushed, the CI script in Gitlab will dynamically generate these .JSON files by replacing the `__underscores__` with their respective values and upload them to AWS.  The script contains a sleep command because AWS has a requirement that the update / create commands can not be used once per second.

## Getting Started: Hello, World!

1. The template name is the filename.  Create two separate files:
   * `hello-world.html`
   * `hello-world.txt`
2. The subject is contained in the first line of the `.html` file.  Continuing the above example:
```
<!-- Subject: Hello World Template -->
<html lang='en'>
    ...
</html>
```
3. Add some content to your `.html` file:
```
<!-- Subject: Hello World Template -->
<html lang='en'>
<body>
    <h1>Hello, World!</h1>
    <p>Lorem ipsum dolor sit amet.</p>
    <h4>&copy; 2022 Rumble Entertainment</h4>
</body>
</html>
```
4. Add some content to your `.txt` file:
```
Hello, World!

Lorem ipsum dolor sit amet.

Copyright 2022 Rumble Entertainment
```
5. Commit and push the repo.
6. On the left hand menu in gitlab, click CI/CD -> Pipelines -> {blocked button} to see the deployment progress.

If the CI script is successful (`.gitlab-ci.yml`), changes are deployed to dev automatically.  Other environments will be listed on this page, but require manual action to run. You can check the script output to ensure the template matches your expectations.

## Next steps: Greeting a particular player

Of course, a template is only as useful as the replacements we can make to it.  The standard way platform makes its replacements is by looking for values encapsulated in curly brackets, but these will vary depending on each template.

1. Start a conversation with the developer implementing the emails on the Platform side.
2. Let them know what replacements you need.
3. Add those replacement placeholders to the templates (in both the HTML and text files):

```
<!-- Subject: Hello, {firstName}! -->
<html lang='en'>
<body>
    <h1>Hello, {fullName}!</h1>
    <p>Lorem ipsum dolor sit amet.</p>
    <a href='{endpoint}'>Acknowledge this email</a>
    <h4>&copy; 2022 Rumble Entertainment</h4>
</body>
</html>
```
4. Push your changes; the Platform dev will need to update the respective endpoints to include these changes.

## Platform Support

The DMZ is responsible for all outgoing emails and requests coming in from emails (if you're providing links).  For organizational purposes, each Platform service should have its own helper class responsible for sending emails.  For example, DMZ -> Utilities -> PlayerServiceEmail.cs contains all of the code tasked with sending the RumbleAccountLogin emails.

The process for adding more emails is:

1. Create a method for each template, along with its replacements.  Continuing the Hello, World example above:
```
public static class FictionalServiceEmail
{
    public const string TEMPLATE_HELLO_WORLD = "hello-world";
    
    public static void SendHelloWorld(string email, string firstName, string fullName, string acknowledgeUrl) =>
        AmazonSes.SendEmail(email, TEMPLATE_HELLO_WORLD, replacements: new RumbleJson
        {
            { "firstName", firstName },
            { "fullName", fullName },
            { "endpoint", acknowledgeUrl }
        }).Wait();
}
```
2. Create an endpoint in the respective DMZ Controller that calls this method, and the acknowledge endpoint:
```
[Route("fiction"), RequireAuth(AuthType.ADMIN_TOKEN)]
public class FictionalController : DmzController
{
    [HttpPost, Route("helloWorld")
    public ActionResult SendHelloWorld()
    {
        string email = Require<string>("email");
        string firstName = Require<string>("firstName");
        string fullName = Optional<string>("fullName") ?? firstName;
        string url = PlatformEnvironment.Url("/dmz/fiction/acknowledge");
        
        FictionalServiceEmail.SendHelloWorld(email, firstName, lastname, url);
        
        return Ok();
    }
    
    [HttpGet, Route("acknowledge"), NoAuth]
    public ActionResult AcknowledgeHelloWorld() => Ok(new RumbleJson
    { 
        { "message", "Thanks for clicking the link!" } 
    });
}
```
3. In your Platform project, hit the DMZ endpoint with the `ApiService` to send out the email:

```
...
    string email = "joe.mcfugal@rumbleentertainment.com";
    string firstName = "Joe";
    string lastName = "McFugal";

    _apiService
        .Request(PlatformEnvironment.Url("/dmz/fiction/helloWorld)
        .AddAuthorization(_config.AdminToken)
        .SetPayload(new RumbleJson
        {
            { "email", email },
            { "firstName", firstName },
            { "fullName", ($"{firstName} {lastName}").Trim() }
        })
        .OnFailure(response => Log.Error(Owner.Default, "Unable to send Hello World Email", data: new
        {
            Response = response.AsRumbleJson
        }));
        .Post(out RumbleJson response, out int code);
...
```


## Restrictions

There are a couple of hiccups to be aware of when working in these templates:

1. The tilde (`~`) is currently used as a delimiter when the CI script is making replacements.  Consequently, using a tilde in any 
2. While the CI script does replace double quotes in the files with single quotes, it hasn't been thoroughly tested any may be on the brittle side.
3. Deleting templates requires using the CLI in your own terminal.  Once a template is uploaded, it can't be removed, even if you delete the file from the repo.
4. AWS SES does not let us create / update / delete templates more than once per second, so the CI script by necessity has sleep times in the loops used to update everything.

## Future Improvements

* Bulletproofing the CI script is a must for long term viability.
  * Using something less restrictive than `sed` for replacements would be great.
* The CI script should eventually delete templates that _aren't_ found in this repo.
* Adding a final step that can be used to delete existing templates may be nice, though should only be rarely used for debugging purposes.
* Finding a way to manage the text content from within the `.html` file would be nice.  Ideally we only have to open one file to maintain a full template.