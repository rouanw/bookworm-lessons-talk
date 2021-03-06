---
title: Puku - lessons learned
---

<!-- Compile using  mdpress -a -s obtvse puku.md -->

<style>
	img{
		height: 550px;
		margin: 20px;
	}

	code{
		font-size: 20px;
	}

	.corner-box{
		text-align: right;
		position: fixed;
		right: 0;
		bottom: 0;
		background-color: rgb(225,225,225);
		padding: 20px;
		border-radius: 15px;
		border: 1px solid rgb(200,200,200);
	}

	.corner-box img{
		height: 50px;
	}
</style>

##Web App Dos and Don'ts
###Lessons from the Bookworm project

<div class="corner-box">
	<p><a href="http://www.thoughtworks.com" ><img src="tw-logo.png"/></a></p>
	<p>Rouan Wilsenach</p>
	<p><a href="https://twitter.com/rouanw">@rouanw</a></p>
</div>

---
##What we'll cover

* Credentials in the source code
* Defect: 500 on some books
* Defect: 500 instead of 404
* Unfriendly error pages
* Hard coded host address

---
##Credentials in the source code
---
Spot the whoopsie


	public void SendConfirmation(string from, string to, string securityToken, int userId)
	{
	    var client = new SmtpClient
	        {
	            Host = "smtp.gmail.com",
	            Port = 587,
	            EnableSsl = true,
	            Credentials = new NetworkCredential("pukusite@gmail.com", "b00ksRc00l")
	        };

	    var mm = new MailMessage(from, to, ConfirmationEmailSubject, string.Format(Template, userId, securityToken))
	        {
	            BodyEncoding = Encoding.UTF8,
	            DeliveryNotificationOptions = DeliveryNotificationOptions.OnFailure
	        };
	    client.Send(mm);
	}

---
Step 1 : Test the email service

---
SmtpClientWrapper - for testing

	public class SmtpClientWrapper
	{
	    public virtual void Send(MailMessage mailMessage, string host, int port, bool enableSsl,
	        NetworkCredential networkCredential)
	    {
	        var smtpClient = new SmtpClient
	        {
	            Host = host,
	            Port = port,
	            EnableSsl = enableSsl,
	            Credentials = networkCredential
	        };

	        smtpClient.Send(mailMessage);
	    }
	}

---
Example test

	  [TestMethod]
	  public void ShouldSendAnEmailUsingSmtpClientWrapper()
	  {
	      _emailService.SendConfirmation("from@thoughtworks.com", "to@thoughtworks.com", "security", 1);

	      _smtpClientWrapper.Verify(it => it.Send(It.IsAny<MailMessage>(), "smtp.gmail.com", 587, true,
	          It.IsAny<NetworkCredential>()));
	  }

---
The new version

	public EmailService(SmtpClientWrapper smtpClient = null, ConfigurationService configurationService = null)
	{
	    _smtpClient = smtpClient ?? new SmtpClientWrapper();
	    _configurationService = configurationService ?? new ConfigurationService();
	}

	public void SendConfirmation(string from, string to, string securityToken, int userId)
	{
	    var mailMessage = new MailMessage(from, to, ConfirmationEmailSubject, string.Format(Template, userId, securityToken))
	        {
	            BodyEncoding = Encoding.UTF8,
	            DeliveryNotificationOptions = DeliveryNotificationOptions.OnFailure
	        };
	    var networkCredential = new NetworkCredential(_configurationService.GetEmailSenderAddress(),
	        _configurationService.GetEmailSenderPassword());
	    _smtpClient.Send(mailMessage, "smtp.gmail.com", 587, true, networkCredential);
	}

---
The configuration service - integration tests

	[TestMethod]
	public void ShouldFetchEmailSenderAddressFromWebConfig()
	{
	    var configService = new ConfigurationService();
	    configService.GetEmailSenderAddress().Should().Be("test-address-value");
	}

	[TestMethod]
	public void ShouldFetchEmailSenderPasswordFromWebConfig()
	{
	    var configService = new ConfigurationService();
	    configService.GetEmailSenderPassword().Should().Be("test-password-value");
	}

---
The config

	&lt;appSettings&gt;
		...
		&lt;add key="emailSenderAddress" value="test-address-value"/&gt;
		&lt;add key="emailSenderPassword" value="test-password-value"/&gt;
	&lt;/appSettings&gt;

---
Config on AppHarbor

![Snippet](email_config.png)

---
But now we've broken the dev environment!

---
web.config

	&lt;appSettings&gt;
		...
		&lt;add key="emailServerAddress" value="localhost"/&gt;
		&lt;add key="emailEnableSsl" value="false"/&gt;
	&lt;/appSettings&gt;

---
![Snippet](papercut.png)


---
## What we've learned

* Use configuration variables for credentials
* Don't use the same password for everything
* Be careful not to send real emails from dev/test environment
* Have a staging email account

---
##Defect: 500 on some books

---
Error details shown on AppHarbor

		Time
		12/23/13 10:14 AM

		Request path
		/Books/4449/Louba-sebapadi-sa-bolo-se-senyenyane

		Commit id
		ab8e9c57156fecbec47e5e0dc9e20fc0d5c8dad5

		Message
		An unhandled exception has occurred.

		Exceptions
		[ArgumentOutOfRangeException: Length cannot be less than
		 zero. Parameter name: length]
		   at System.String.InternalSubStringWithChecks(Int32 startIndex, Int32 length, Boolean fAlwaysCopy)
		   at BookWorm.ViewModels.BookInformation.Summary(Int32 characters) in d:\temp\wqfl5nkd.eis\input\BookWorm\ViewModels\BookInformation.cs:line 100
		   at BookWorm.Controllers.BooksController.Details(Int32 id) in 

---
The method throwing the exception

		public string Summary(int characters)
		{
		    if (Model.Description == null || Model.Description.Length < characters)
		        return Model.Description;
		    return Model.Description.Substring(0, Model.Description.IndexOf(" ", characters));
		}

---
No tests. Lets add some. #1

    [TestMethod]
    public void SummaryShouldReturnBookDescriptionIfItsNull()
    {
        var bookInformation = new BookInformation
        {
            Model = new Book
            {
                Description = null
            }
        };
        bookInformation.Summary(150).Should().Be(null);
    }
---
Test #2

    [TestMethod]
    public void SummaryShouldReturnBookDescriptionWhenItsShorterThanTheSummaryLengthRequested()
    {
        var bookInformation = new BookInformation
        {
            Model = new Book
            {
                Description = "four"
            }
        };
        bookInformation.Summary(5).Should().Be("four");
    }

---
Test #3

	[TestMethod]
	public void SummaryShouldReturnTheBookDescriptionUpToTheFirstSpaceAfterTheCharacterLimitRequested()
	{
	    var bookInformation = new BookInformation
	    {
	        Model = new Book
	        {
	            Description = "word word"
	        }
	    };
	    bookInformation.Summary(2).Should().Be("word");
	}

---
Now our failing test

	[TestMethod]
	public void SummaryShouldReturnTheBookDescriptionTruncatedToTheCharacterLimitIfThereAreNoSpacesAfterTheLimit()
	{
	    var bookInformation = new BookInformation
	    {
	        Model = new Book
	        {
	            Description = "wordword"
	        }
	    };
	    bookInformation.Summary(4).Should().Be("word");
	}

---
Make the code pass

![Snippet](fix_slug_bug.png)

---
## What we've learned

* Monitor application logs for exceptions
* Make sure the code you're changing is tested
* Write a failing test for the defective case

---

## Defect: 500 instead of 404
www.puku.co.za/Books/1234

<!---
https://github.com/ThoughtWorksZA/bookworm/commit/8b6b60563b950699d400803191494fdc09b2f5d3
-->
---
Code with bug

	[AllowAnonymous]
	public ViewResult Details(int id)
	{
	    var book = Repository.Get<Book>(id);
	    var bookInformation = new BookInformation(book, book.Posts.Select(post => new BookPostInformation(book.Id, post)).ToList());
	    ViewBag.Title = bookInformation.Model.Title;
	    ViewBag.MetaDescription = bookInformation.Summary(155);
	    return View(bookInformation);
	}
---
Failing test

	[TestMethod]
	public void DetailsShouldThrowA404ExceptionWhenBookIsNotFound()
	{
	    var repo = new Mock<Repository>();
	    repo.Setup(it => it.Get<Book>(1)).Returns((Book)null);
	    var booksController = new BooksController(repo.Object);
	    
	    Action getDetails = () => booksController.Details(1);

	    getDetails.ShouldThrow<HttpException>().WithMessage("The requested book could not be found").Where(it => it.GetHttpCode() == 404);
	}

---
Make it pass

	[AllowAnonymous]
	public ViewResult Details(int id)
	{
	    var book = Repository.Get<Book>(id);

	    if (book == null)
	    {
	        throw new HttpException(404, "The requested book could not be found");
	    }

	    var bookPosts = book.Posts.Select(post => new BookPostInformation(book.Id, post)).ToList();
	    var bookInformation = new BookInformation(book, bookPosts);
	    ViewBag.Title = bookInformation.Model.Title;
	    ViewBag.MetaDescription = bookInformation.Summary(155);
	    return View(bookInformation);
	}

---
## What we've learned

* Be wary of null / empty returns from methods
* Keep HTTP standards in mind
* Write a failing test for the defective case

---
##Unfriendly error pages

---
![Snippet](bad404.png)

---
Custom error pages - web.config

	&lt;system.web&gt;
		&lt;customErrors mode="RemoteOnly" defaultRedirect="/InternalError.html"&gt;
			&lt;error statusCode="404" redirect="/NotFound.html" /&gt;
			&lt;error statusCode="500" redirect="/InternalError.html" /&gt;
		&lt;/customErrors&gt;
		...
 	&lt;/system.web&gt;

---
![Snippet](404.png)

---
## What we've learned

* Be nice to your users
* Make sure your app can deal with problems gracefully

---
## Hard coded host address
---
Don't hard code host in emails to users

- Open source
- Testing

<!--
https://github.com/ThoughtWorksZA/bookworm/commit/07a06dec17b860b2f5e3a82517b02c9502504465
https://github.com/ThoughtWorksZA/bookworm/commit/070da901eb7a3620514109d09d26f15731ddf5a7
-->
---
EmailService to get host from CurrentHttpContextWrapper

	[TestMethod]
	public void ShouldGetBaseUrlFromCurrentHttpContextWrapper()
	{
	    _currentHttpContextWrapper.Setup(it => it.GetBaseUrl()).Returns("someUrl");
	    _emailService.SendConfirmation("from@thoughtworks.com", "to@thoughtworks.com", "security", 1);
	    _smtpClientWrapper.Verify(it => it.Send(It.Is<MailMessage>(mail => mail.Body.Contains("someUrl/Users/1/RegisterConfirmation/security")),
	        It.IsAny<string>(), It.IsAny<int>(), It.IsAny<bool>(), It.IsAny<NetworkCredential>()));
	}

---
The ContextWrapper

	public class CurrentHttpContextWrapper
	{
	    public virtual string GetBaseUrl()
	    {
	        var urlFormattingHelper = new UrlFormattingHelper();
	        return urlFormattingHelper.GetBaseUrl(System.Web.HttpContext.Current.Request.Url);
	    }
	}

---
Put the logic somewhere you can test

 	[TestClass]
	class UrlFormattingHelperTests
	{
	    [TestMethod]
	    public void ShouldCombineSchemeAndAuthority()
	    {
	        var helper = new UrlFormattingHelper();
	        var baseUrl = helper.GetBaseUrl(new Uri("http://localhost:1234/soup"));
	        baseUrl.Should().Be("http://localhost:1234");
	    }
	}

---
But AppHarbor does load balancing...

http://puku-staging.apphb.com:16140/Users/321/RegisterConfirmation/...

---
Web.config

	&lt;appSettings&gt;
	 ...
	 &lt;add key="aspnet:UseHostHeaderForRequestUrl" value="true" /&gt;
	 ...
	&lt;/appSettings&gt;

---
## What we've learned

* Don't hard code things that can change without breaking tests
* Keep potential open source users in mind
