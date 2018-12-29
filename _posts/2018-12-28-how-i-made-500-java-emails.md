---
layout: post
title:  How I made $500 in a few hours by sending automated e-mails with Java
author: warren
image: assets/images/varo.png
---
Recently, I saw an interesting bank promotion from Varo Money on <a href="https://www.doctorofcredit.com/varo-money-online-bank-account-100-sign-up-bonus-ios-only/" target="_blank">Doctor of Credit</a>. The terms were fairly easy to meet, requiring only a single direct deposit of $250. **Far more interesting, however, was Varo's referral program which promised a bonus of $100 for every user who signed up via my referral link and set up direct deposit (up to $500).** On the Doctor of Credit <a href="https://www.doctorofcredit.com/varo-money-referral-post/" target="_blank">blog post comments</a>, many people were asking for referrals to Varo, but there were equally as many people (or more) with referrals to give out. To maximize the chance of people using my referral link, I turned to Java and automation. I wrote a quick Java script that would poll the Doctor of Credit comments section and instantly send an e-mail with my referral link to anyone who asked. **Within hours, I was $500 richer.**<br/><br/>In this post, I'll walk through the code which earned me $500, with a link to the entire program on GitHub at the end of the article.
## Solution
> Note: To preserve space, code samples below might be incomplete. To see the complete program, check out the GitHub link in the Conclusion.

* Subscribed to Doctor of Credit comments via e-mail to instantly get notified whenever someone posted a new comment.
![subscribe]({{ site.baseurl }}/assets/images/subscribe.png)

* Used the <a href="https://javaee.github.io/javaee-spec/javadocs/javax/mail/package-summary.html" target="_blank">JavaMail API</a> to connect to my Gmail account.

```java
logger.info("Connecting to inbox");
Session session = Session.getInstance(props,
		new javax.mail.Authenticator() {
			protected PasswordAuthentication getPasswordAuthentication() {
				return new PasswordAuthentication(props.getProperty("email.address"), props.getProperty("email.password"));
            }
		});
Store store = null;
Folder inbox = null;
try {
	store = session.getStore(props.getProperty("protocol"));
	store.connect(props.getProperty("mail.smtp.host"), props.getProperty("email.address"), props.getProperty("email.password"));
	inbox = store.getFolder("inbox");
	inbox.open(Folder.READ_WRITE);
} catch (MessagingException e) {
	logger.error("Could not create or connect to store/inbox. Exception: ", e);
}
```

* **Each minute**, I check for new e-mails in my Gmail inbox. If there's an e-mail from the Varo comments subscription, I try and extract an e-mail address from the e-mail body by using regex.

```java
logger.info("Iterating through unseen messages");
// search for all "unseen" messages
Flags seen = new Flags(Flags.Flag.SEEN);
FlagTerm unseenFlagTerm = new FlagTerm(seen, false);
Message messages[] = null;
try {
	messages = inbox.search(unseenFlagTerm);
} catch (MessagingException e) {
	logger.error("Could not retrieve messages. Exception: ", e);
}

if (messages.length == 0) {
	logger.info("No messages found");
};

try {
	for (int i = 0; i < messages.length; i++) {
		if(messages[i].getSubject().contains("New Comment On Varo Money Referral Post")) {
			List<String> emailAddresses = null;
			try {
				emailAddresses= getEmailAddresses(getText(messages[i]));
			} catch (MessagingException | IOException e) {
				logger.error("Could not get email addresses. Exception: ", e);
			}
			sendEmails(emailAddresses, session, props);
		}
		messages[i].setFlag(Flag.SEEN, true);
	}
} catch (MessagingException e) {
	logger.error("Could not iterate through messages. Exception: ", e);
}
```

```java
private static List<String> getEmailAddresses(String text) {
	List<String> emailAddresses = new ArrayList<>();
	Matcher m = Pattern.compile("[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+").matcher(text);
	while (m.find()) {
		emailAddresses.add(m.group());
	}
	return emailAddresses;
}
```

* I now have the e-mail address of the person who wanted a referral, so I send them an e-mail that contains my Varo referral link.

```java
private static void sendEmails(List<String> emailAddresses, Session session, Properties props) {
	logger.info("Sending email");
	if (emailAddresses.size() > 0) {
		Message message = new MimeMessage(session);
		try {
			message.setFrom(new InternetAddress(props.getProperty("email.address")));
			message.setRecipients(Message.RecipientType.TO, InternetAddress.parse(emailAddresses.get(0)));
			message.setSubject(props.getProperty("mail.subject"));
			message.setContent(props.getProperty("mail.body"), "text/html; charset=utf-8");
		} catch (MessagingException e) {
			logger.error("Could not create message to send. E-mail address: {} Exception: ", emailAddresses.get(0), e);
		}

		try {
			Transport.send(message);
			logger.info("Email to {} sent",  emailAddresses.get(0));
		} catch (MessagingException e) {
			logger.error("Could not send message. E-mail address: {} Exception: ", emailAddresses.get(0), e);
		} 
	}
}
```
Relevant properties:
```
# Specific for Gmail
mail.smtp.auth=true
mail.smtp.starttls.enable=true
mail.smtp.host=smtp.gmail.com
mail.smtp.port=587

#Properties I defined
email.address=warrencrasta@gmail.com
email.password=XXXXXX
mail.subject=Varo $100 Referral Link
mail.body=Hi,<br/><br/>I saw your post on Doctor of Credit seeking a referral for the Varo $100 bonus.<br/><br/>Here is my referral link: http://refer.varomoney.com/rg9zv.<br/><br/>Would very much appreciate if you could sign up with my link. Please let me know if you use it. Thanks!
protocol=imaps
```
Easy as that! **By sending an e-mail to a person at exactly the same time they asked for a referral link, it was almost guaranteed that they would use my referral as opposed to anyone else's.**
## The Results
![referrals]({{ site.baseurl }}/assets/images/referrals.jpg) 
## Conclusion
The complete code is available on GitHub <a href="https://www.github.com/wcrasta/varo" target="_blank">here</a>. If you found the code useful, **I would really appreciate it if you starred the repo.** Normally, I would've run this script via a cronjob on a remote Linux instance (EC2, Digital Ocean, etc.). However, many providers, including AWS, have blocked outbound mail on the mail ports since people are presumably using it for spam. As a result, I've done a somewhat hack of calling ```Thread.sleep()``` to poll instead of using a cronjob, since I ran this code on my Windows machine. Alternatively, I considered using <a href="https://aws.amazon.com/ses/" target="_blank">Amazon Simple Email Service (Amazon SES)</a> to send mail, but I found that the e-mails I was sending were going to spam.<br/><br/>While the Varo promotion might've ended by the time you're reading the post, I hope you found this post informative and useful for the future. Feel free to e-mail me your comments/feedback at <warrencrasta@gmail.com>, would love to read them. Thanks for reading!
