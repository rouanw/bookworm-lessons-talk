#Fixing some code
##On puku.co.za

---
Blank for now

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

	<appSettings>
	  ...
	  <add key="emailSenderAddress" value="test-address-value"/>
	  <add key="emailSenderPassword" value="test-password-value"/>
	</appSettings>

---
Config on AppHarbor

![Snippet](email_config.png)

---
But now we've broken the dev environment!

---
web.config

  <appSettings>
  	...
    <add key="emailServerAddress" value="localhost"/>
    <add key="emailEnableSsl" value="false"/>
  </appSettings>

---
![Snippet](papercut.png)

---
##Production defect

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

## 500 instead of 404
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
##Unfriendly error pages

---
![Snippet](bad404.png)

---
Custom error pages - web.config

   <system.web>
      <customErrors mode="RemoteOnly" defaultRedirect="/InternalError.html">
        <error statusCode="404" redirect="/NotFound.html" />
        <error statusCode="500" redirect="/InternalError.html" />
      </customErrors>
      ...
	 </system.web>

---
![Snippet](404.png)

---

