# Make XSS Hunter ~~great again!~~ send Slack notifications

0. [Install XSS Hunter](https://thehackerblog.com/xss-hunter-is-now-open-source-heres-how-to-set-it-up/). It's awesome. (Note that you might want to omit MailGun configuration.)

1. Create a Slack bot for your team as described in [this tutorial](https://api.slack.com/bot-users)

1. Add the following to `config.yaml`. Start `send_to` with # to post to channel or with @ to send direct message to a user.

    ~~~
    slack_api_key: <YOUR SLACK BOT API KEY>
    slack_send_to: '#general'
    ~~~

1. Modify `api/appserver.py` as shown below. Note that `send_slack` definition and call are related to notifications, the rest is security and functionality tweaks.

    ~~~
    diff --git a/api/apiserver.py b/api/apiserver.py
    index 336c09f..63c74c4 100755
    --- a/api/apiserver.py
    +++ b/api/apiserver.py
    @@ -26,6 +26,8 @@ from models.request_record import InjectionRequest
     from models.collected_page import CollectedPage
     from binascii import a2b_base64

    +from slacker import Slacker
    +
     logging.basicConfig(filename="logs/detailed.log",level=logging.DEBUG)

     try:
    @@ -143,6 +145,8 @@ class BaseHandler(tornado.web.RequestHandler):
             domain = self.request.headers.get( 'Host' )
             domain_parts = domain.split( "." + settings["domain"] )
             subdomain = domain_parts[0]
    +       if subdomain == settings["domain"]:
    +               subdomain = "xssl"
             return session.query( User ).filter_by( domain=subdomain ).first()

     def data_uri_to_file( data_uri ):
    @@ -247,6 +251,26 @@ def send_email( to, subject, body, attachment_file, body_type="html" ):
                 auth=("api", settings["mailgun_api_key"] ),
                 callback=email_sent_callback)

    +def send_slack(injection_db_record):
    +    slack = Slacker(settings['slack_api_key'])
    +    slack_send_to = settings['slack_send_to']
    +    injection_data = injection_db_record.get_injection_blob()
    +    slack_message = '*XSS Payload Fired On* ' + injection_data['vulnerable_page'] + '\n\t*IP:* ' + injection_data['victim_ip']
    +    if injection_data['referer']:
    +       slack_message = slack_message + '\n\t*Referer:* '+ injection_data['referer']
    +    slack_message = slack_message + '\n\t*User Agent:* ' + injection_data['user_agent']
    +    if injection_data['cookies']:
    +       slack_message = slack_message + '\n\t*Cookies:* ' + injection_data['cookies']
    +    slack_message = slack_message + '\n\t*DOM:*\n```' + injection_data['dom'] + '```'
    +    slack_message = slack_message + '\n\t*Origin:* ' + injection_data['origin']
    +    slack_message = slack_message + '\n\t*Screenshot:* https://' + settings['domain'] + '/' + injection_data['screenshot']
    +    if slack_send_to[0] == '#':
    +       slack.chat.post_message(slack_send_to, slack_message)
    +    elif slack_send_to[0] == '@':
    +       slack.chat.post_message(slack_send_to, slack_message, as_user=True)
    +    else:
    +       slack.chat.post_message('#general', slack_message)
    +
     def send_javascript_pgp_encrypted_callback_message( email_data, email ):
         return send_email( email, "[XSS Hunter] XSS Payload Message (PGP Encrypted)", email_data, False, "text" )

    @@ -408,6 +432,7 @@ class CallbackHandler(BaseHandler):
                 injection_db_record = record_callback_in_database( callback_data, self )
                 self.logit( "User " + owner_user.username + " just got an XSS callback for URI " + injection_db_record.vulnerable_page )

    +           send_slack(injection_db_record)
                 if owner_user.email_enabled:
                     send_javascript_callback_message( owner_user.email, injection_db_record )
                 self.write( '{}' )
    @@ -681,5 +706,5 @@ if __name__ == "__main__":
         tornado.options.parse_command_line(args)
         Base.metadata.create_all(engine)
         app = make_app()
    -    app.listen( 8888 )
    +    app.listen( 8888, address="127.0.0.1" )
         tornado.ioloop.IOLoop.current().start()
    ~~~

1. Turn off email notifications in MailGun if you configured it.