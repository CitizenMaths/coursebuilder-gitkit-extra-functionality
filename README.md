# coursebuilder-gitkit-extra-functionality
Additional functionality added to Gitkit code to allow transfer of progress and personal data from a pre-exisiting user using a Google sign in account to a Gitkit account, as well as storing the user temporarily during the life of a single request to save making multiple lookups to the gitkit service

###Note:

1: The code that copies data from one account to another came about because when we decided to implement gitkit from the google sign in only, so a user did not have to use a google associated email address to login,
we found that the two accounts (google sign in only and gitkit) used very different user id's for their implementation, and thus they would have to register again and all their progress would have been lost.
The code attempts to find the old account, if there is one, and copy their personal and progress data from the google sign in account to the new gitkit account, so a user would not lose any progress that
has been made throughout the course.

2: The code that stores the current user for the duration of a request came about because of course content pages being extremely slow to load.
It was because so many calls to the get_current_user was happening for each request, and everytime it would have to go to the gitkit service and look them up again.
The current user gets stored in a local thread, like the current token does, for the duration of a single request.
It still has to look up the user from the gitkit service, but only once per request now instead of multiples. The user that is looked up then gets stored in the local thread,
and everytime get_current_user is called, it checks the local thread before making a call to the gitkit service to look them up.


###Important:

You will need to setup coursebuilder to use GITKIT before any of this code is useful.
You can find instructions on how to setup GITKIT in the Google's own documentation,
and this guide may also prove useful in setting coursebuilder up to use GITKIT: https://github.com/pabloppp/Course-Builder-gitkit-setup-guide
Even though the guide says it is for GCB 1.9, the majority is still valid for GCB 1.10


###Modifications to the code at modules/gitkit/gitkit.py

###Additional static variables:

Line #236 - `_CM_NAMESPACE = '<YOUR_COURSE_NAMESPACE_HERE>'`
You need to add your course namespace here so the code accesses the correct datastore

Line #237 - `_COURSE_COMPLETION_PROPERTY_NAME = 'linear-course-completion'`

Line #238 - `_SKILL_COMPLETION_PROPERTY_NAME = 'skill-completion'`

These are for copying and inserting progress data from one account to another


###Additional methods added:

Line #336 - `is_gitkit_user_registered` - checks if a gitkit user id is already registered in the course datastore or not

Line #345 - `find_gae_user` - attempts to find an old user account with a given email address to see if we know about an account that was registered when the course only used google sign in only

Line #358 - `copy_gae_user` - copies the personal data from the Student record from the old account into a newly created gitkit account with a new record in the Student datastore

Line #376 - `copy_gae_user_completion_info` - copies the progress data from the old account to the new account so a user does not lose progress the next time they login

Line #395 - `copy_gae_user_answers` - copies any answers to quizzes or questions a user may have done on their old account to their new account


Line #854 - `get_current_user` - gets the current user stored in the local thread, returns None if no current user

Line #861 - `set_current_user` - sets the current user in the local thread


###Additional Code to pre-existing methods/classes:

Lines #312-323 in method apply(cls, user) in class EmailUpdatePolicy

`_LOG.info('gitkit user:')
_LOG.info(student)
if cls.is_gitkit_user_registered(user_id) == False:
    gae_student = cls.find_gae_user(new_email)
    if gae_student is not None:
        _LOG.info('gae user:')
        _LOG.info(gae_student.additional_fields)
        gitkit_student = cls.copy_gae_user(user_id, new_email, gae_student)
        cls.copy_gae_user_completion_info(gae_student, gitkit_student)
        cls.copy_gae_user_answers(gae_student.user_id, user_id)
    else:
        _LOG.info('user already registered')`

		
The above segment of code is the entry point into the copying of account and progress data from a google sign in only account to a gitkit account.

It is processed on a gitkit sign in.

It first checks if the user_id is already registered in the course datastore. If it is, it does nothing else and continues on with the sign in.

If it doesn't, then it checks if the gitkit sign in is using an email that we already know about from a google sign in account.

If it doesn't find any old account with that email, it continues with sign in and takes them to register page.

If it does, then the personal and progress data is copied from the old account to the new gitkit account. It then proceeds with the sign in.


Line #841 in class Runtime

`_threadlocal.current_user = None`

Sets up a current_user variable for the local thread and initialised to None


Line #817 in method __enter__ in class RequestContext

`Runtime.set_current_user(None)`


Line #827 in method __exit__ in class RequestContext

`Runtime.set_current_user(None)`


The two above lines ensure that at the beginning and end of a request that the current_user is set to None to ensure that another user is not carried over from another request.


Lines #1098-1100 in method get_current_user in class UsersService

`rtu = Runtime.get_current_user()
if rtu is not None:
    return rtu`


The above code checks if there is a current user stored in the local thread for the request. If there is, it returns the user and the method breaks off here.


Lines #1129-1134 in method get_current_user in class UsersService

`gu = service.get_user(token)
if gu is not None:
    Runtime.set_current_user(gu)

commented out: #return service.get_user(token)
return gu`


The above code segment replaces the original bit which you can see commented out.

The returned user from the gitkit service is stored in gu and if it exists, then we store it in the local thread for the duration of the request.

Finally, gu is returned by the method rather than service.get_user(token) as we already called that method and stored it in the aforementioned variable gu.


Lines #1349-1356 in method get in class SignInContinueHandler

`tmp = self._get_redirect_url_from_dest_url()
if 'course.citizenmaths.com/main' in tmp or 'citizenmaths-phase-2.appspot.com/main' in tmp:
    dest = 'https://course.citizenmaths.com/main/register'
else:
    dest = self._get_redirect_url_from_dest_url()

commented out: #self.redirect(self._get_redirect_url_from_dest_url())
self.redirect(dest)`


The above code checks whether they are logging into the course, and if so re-directs them to the register page rather than the course homepage.
If a user that is logging in is already registered, it will automatically re-direct them to the course homepage.
