The KarmaNotes Django Experiment
================================

Dependencies
------------

 + Python2.7
 + django1.4
 + Solr3.6.0
 + django-celery2.5.5


To install python package dependencies run:

    pip install -r requirements.txt


Deployment
----------

1. checkout the git repository, we use `/var/www/djKarma` and will refer to this as the root of the repo
2. install requirements with `sudo pip install -r requirements.txt`
3. Create the database if a new deployment with `./manage.py syncdb` If this is not a new deployment, see the section below on database migrations. NOTE: You can't create a superuser BEFORE loading the fixtures.
4. Use south to migrate `djcelery` and `kombu.transport.django`:
    ./manage.py migrate djcelery
    ./manage.py migrate kombu.transport.django
5. Create superuser. `./manage.py createsuperuser
6. Start celery task server (see Note)
7. Start Apache Solr search engine (see section)

 + TODO: short desc of how to install and deploy on a deployment server, what server packages need to be running/installed, but not how to install them
### Future

8. copy $(djKarma_src)/bin/knotes -> /etc/init.d/knotes . This is the init script for running karmanotes at startup. Please
note that the stop / kill options do not work because of pid issues.  A fix is being worked on.

### Deployment database
To initially deploy the postgres backup on a fresh debian based system:
```
sudo apt-get install postgresql-9.1 python-psycopg2
sudo passwd postgres
sudo su postgres
sudo -u postgres createuser -P djkarma
psql template1
create database karmanotes owner djkarma encoding 'UTF8';
#### add this line to your postgres install's /etc/postgresql/9.1/main/pg_hba.conf ####
local   karmanotes      djkarma                                 md5
sudo service postgresql restart
./manage.py syncdb # if you do not do this `run_gunicorn` will not be a command option
```
### Search: Apache Solr 3.6.0

#### Installation

see the [official Solr tutorial](http://lucene.apache.org/solr/api-3_6_0/doc-files/tutorial.html).


#### Maintaining the Search Index

see the [official Solr tutorial](http://lucene.apache.org/solr/api-3_6_0/doc-files/tutorial.html). Also the [Django Haystack note](http://django-haystack.readthedocs.org/en/v1.2.7/tutorial.html#reindex) on maintaining the index.

When changes are made to the search index schema (./notes/search_indexes.py), the index must be rebuilt (similiar to models.py and database migrations).


##### Rebuilding the search schema AND index after modifying ./notes/search_indexes.py:

1. **Build the Index Schema**. You can generate the schema from the django application (once Haystack is installed and setup) by running:

		 ./manage.py build_solr_schema > /path/to/solr/conf/schema.xml
		 
2. **Restart Solr**
3. **Build the Search Index**. A shortcut for clear_index followed by update_index. 

		./manage.py rebuild_index

##### Updating the search index after new models are added
If you've added new models in Django, and haven't touched search_index.py:

		./manage.py update_index

###### Note on Solr location: 
	
With Solr installed using Homebrew on Mac OS X, the default path will be:
        
        /usr/local/Cellar/solr/3.6.0/libexec/example/solr/conf/schema.xml
		 
On the current server, the path to the solr schema is:

		/home/dbro/apache-solr-3.6.0/example/solr/conf/schema.xml


### Celery Task Server


The celery task server asynchronously handles the Google Documents API processing so the main django server remains responsive. When a document is uploaded via /upload, the django app (specifically the custom django-ajax-uploader backend) will ask the celery server to upload the document (now on the local server) to Google and await their response. The code which the celery server executes is actually located in ./notes/tasks.py

TODO: The new upload process works in two requests. One for the actual file, and a second for the file's meta data. If a file is processed but no meta data is submitted, the file needs to be marked specially. Currently, this behavior allows files without any meta data to appear wherever the site's files are displayed. Secondly, we need to create an avenue for a user to re-visit these incomplete uploads and enter meta data at their convenience. Only when this has happened should the files be displayed as normal.

To install and run the celery task server:

1. Ensure django-celery is installed per the requirements
2. Place the celery init.d script (./packaging/celeryd) in /etc/init.d/
3. Place the celery config file (./packaging/celeryconfig) in /etc/default
4. Make an unprivileged, non-password-enabled user and group to run celery
        useradd celery
        
5. make a spot for the logs and the pid files
        mkdir /var/log/celery
        mkdir /var/run/celery
        chown celery:celery /var/log/celery
        chown celery:celery /var/run/celery
6. chmod +x /etc/init.d/celeryd
7. run /etc/init.d/celeryd start

###Production Note:
On the new Amazon instance, celery logs are located at:

	/run/celery



Note on Database migrations
---------------------------

I've installed South to help ease database migrations whenever the models.py file is altered.
South's workflow is as follows for changes to the 'notes' app:

To initialize South:

New project:

1. ./manage.py schemamigration notes --initial

Existing project:

1. ./manage.py convert_to_south notes


To perform migrations with South:

1. Alter models.py as needed
2. ./manage.py schemamigration notes --auto
3. ./manage.py migrate notes

Note on Creating a Development DB from Production DB
----------------------------------------------------

If you'd like to copy the current production database and use it during your development:

### On Production Machine:

1. ssh into production project root
2. Note the git state of the production project (with git log or likewise). NOTE: ignore merges performed by production system users (Commits where Author is a system username not a GitHub username).
3. python manage.py dumpdata notes --exclude notes.userprofile > ~/dumps.json. Transfer dumps.json to development machine.

### On Development Machine:

1. checkout the commit corresponding to the production project state
2. Make sure south is commented out in settings.INSTALLED_APPS
3. manage.py syncdb (do NOT create superuser when prompted)
		
	+ If syncdb loads fixtures: 
	    a. manage.py createsuperuser
	+ Else:
		a. manage.py loaddata ./path/to/fixtures.json 
		    + *Note:* fixtures.json was initial_data.json earlier in the project history. 
		b. Then manage.py createsuperuser
4. manage.py loaddata ~/dumps.json 

5. Now un-comment south from settings.INSTALLED_APPS and manage.py syncdb
6. manage.py convert_to_south notes
7. git pull origin master
8. manage.py schemamigration notes --auto
9. if schemamigration prompts you for a default value. i.e: 
	
		Added field owner on notes.File
		? The field 'File.type' does not have a default specified, yet is NOT NULL.
 		? Since you are making this field nullable, you MUST specify a default
 		? value to use for existing rows. Would you like to:
 		?  1. Quit now, and add a default to the field in models.py
 		?  2. Specify a one-off value to use for existing columns now
 		?  3. Disable the backwards migration by raising an exception.
 		? Please select a choice: 2
 	
 	Simply enter 2 and enter a one-off value (In this case 'N' for Note) if you understand the data model. Else specify 3. In development, losing the ability to backwards migrate is not a big deal.

10. manage.py migrate notes


Note on Google Documents API
----------------------------

 To authorize the Google Documents script to act on behalf of the target google account, verfication will have to be granted by visiting:

 https://accounts.google.com/DisplayUnlockCaptcha

 In my testing I haven't yet had the oauth token expire, but I'll have to investigate further.


Style compilation
-----------------
To recompile css from the less source files, install the latest Node js, which comes with the npm package manager and install 'less'

## TODO automate this process, either with a make script or as a django ./manage.py command

less source files live in `djKarma/static/less` To recompile `style.css`, you must install lessc

    sudo npm install -g less

You will now have the command lessc available to you. run this command from the root of the project repo

    lessc ./static/less/styles.less ./static/css/styles.css
   
  
Management Commands (manage.py \<command name>)
-----------------------------------
+ **default_courses**: Assign default values to Course fields required by Solr indexing, but not enforced by models.py spec.
   + Checks School, Academic Year, Semester fields of all Course objects.
+ **assign_file_owners**: Assign File.owner fields based on User.files and/or a prompted default user.
   + Necessary when a backup of the notes app database is made separate of the User table 

