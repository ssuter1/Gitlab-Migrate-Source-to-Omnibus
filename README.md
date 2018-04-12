# Gitlab-Migrate-Source-to-Omnibus


1. Convert database from MySQL to postgres
Derived from this guide:
https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/install/installation.md#6-database
```
sudo apt-get install -y postgresql postgresql-client libpq-dev postgresql-contrib
sudo -u postgres psql -d template1 -c "CREATE USER git CREATEDB;"
sudo -u postgres psql -d template1 -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
```		
Test connection to new database: 
```
sudo -u git -H psql -d gitlabhq_production
```
Check if the pg_trgm extension is enabled (After logging into postgres db with command above):
```
SELECT true AS enabled
FROM pg_available_extensions
WHERE name = 'pg_trgm'
AND installed_version IS NOT NULL;           (Will return enabled)
\q         
```
Install latest version of "pgloader" package from PostgreSQL repo:
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt-get install wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install pgloader
sudo service gitlab stop
```
Migrate Database from MySQL to PostgreSQL
```
cd /home/git/gitlab
sudo -u git mv config/database.yml config/database.yml.bak
sudo -u git cp config/database.yml.postgresql config/database.yml
sudo -u git -H chmod o-rwx config/database.yml
```
Prepare DB schema (This step had issues)
```
sudo -u git -H bundle exec rake db:create db:migrate RAILS_ENV=production
```
Errors from above command:

Specified 'postgresql' for database adapter, but the gem is not loaded. Add `gem 'pg'` to your Gemfile (and ensure its version is at the minimum required by ActiveRecord). Couldn't create database for {"adapter"=>"postgresql", "encoding"=>"unicode", "database"=>"gitlabhq_production", "pool"=>10}

[2018-04-02T11:27:05.318254 #4985] DEBUG -- sentry: ** [Raven] Specified 'postgresql' for database adapter, but the gem is not loaded. Add `gem 'pg'` to your Gemfile (and ensure its version is at the minimum required by ActiveRecord). excluded from capture due to environment or should_capture callback

Fix:
```
vi /home/git/gitlab/Gemfile 
Commnent out line "#gem 'pg', '~> 0.18.2', group: :postgres"
Add line: "gem 'pg', '~> 0.20'"
"bundle install --no-deployment"
sudo -u git -H bundle exec rake db:create db:migrate RAILS_ENV=production
```
Migrate data from MySQL to PostgreSQL:

Create file /home/git/gitlab/commands.load with contents:

					1. LOAD DATABASE
					2.      FROM mysql://username:password@host/gitlab           (Grab creds form KeePass)
					3.      INTO postgresql://postgres@unix://var/run/postgresql:/gitlabhq_production
					4. 
					5. WITH include no drop, truncate, disable triggers, create no tables,
					6.      create no indexes, preserve index names, no foreign keys,
					7.      data only
					8. 
					9. ALTER SCHEMA 'gitlab' RENAME TO 'public'
					10. 
					11. ;
				2. sudo -u postgres pgloader commands.load
				3. sudo service gitlab start
				4. Pray
	2. Restore converted database to Gitlab Omnibus installation

		1. Temporarily fix permissions on /home/git/gitlab/tmp/backups directory: "chmod -R 777 /home/git/gitlab/tmp/backups"
		2. Create a full backup using new postgres DB: sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
		3. Download identical Omnibus .deb to /opt: wget --content-disposition https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/trusty/gitlab-ce_8.10.13-ce.0_amd64.deb/download.deb
		4. Important or shit will go horribly wrong: Stop all gitlab related processes and prevent them from starting on boot:

			1. sudo update-rc.d gitlab disable
			2. sudo update-rc.d nginx disable
			3. sudo update-rc.d redis-server disable
			4. sudo service gitlab stop
			5. sudo service nginx stop
			6. sudo service redis-server stop
		5. dpkg -i gitlab-ce_8.10.13-ce.0_amd64.deb/download.deb
		6. Once installed, run "gitlab-ctl reconfigure"
		7. Overwrite gitlab.rb file with our own version
		8. Run "gitlab-ctl reconfigure" again.  
		9. Copy full backup taken in step 2 to /var/opt/gitlab/backups "cp 1523306095_gitlab_backup.tar /var/opt/gitlab/backups/"
		10. Stop processes that access the db:

			1. sudo gitlab-ctl stop unicorn
			2. sudo gitlab-ctl stop sidekiq
		11. Fix /var/opt/backups/XXXXXXXXXX_gitlab_backup.tar permissions: "chown git:root /var/opt/gitlab/backups/XXXXXXXXXX_gitlab_backup.tar"
		12. Restore backup to new Omnibus install: "sudo gitlab-rake gitlab:backup:restore BACKUP=XXXXXXXXXX"
		13. Check gitlab database and config files: "sudo gitlab-rake gitlab:check SANITIZE=true"
		14. Hopefully you'll get this result:

			1. Redis version >= 2.8.0? ... yes
			2. Ruby version >= 2.1.0 ? ... yes (2.1.8)
			3. Your git bin path is "/opt/gitlab/embedded/bin/git"
			4. Git version >= 2.7.3 ? ... yes (2.7.4)
			5. Active users: 559
			6. 
			7. Checking GitLab ... Finished
		15. Move contents of /home/git/gitlab/.secret to gitlab_rails secret in /etc/gitlab/gitlab-secrets.json
		16. Move contents of /home/git/gitlab/.gitlab_shell_secret to gitlab_shell in /etc/gitlab/gitlab-secrets.json
		17. Move contents of /home/git/gitlab/config/secrets.yml to db_key_base in /etc/gitlab/gitlab-secrets.json
		18. Run "gitlab-ctl reconfigure"
		19. Move SSL certs from old gitlab install to new Omnibus:

			1. Copy bundle.crt and lcsr-domain.key from /etc/nginx/ssl to /etc/gitlab/ssl
			2. Modify options in /etc/gitlab/gitlab.rb: 

				1. nginx['enable'] = true
				2. nginx['redirect_http_to_https'] = true
				3. nginx['redirect_http_to_https_port'] = 80
				4. nginx['ssl_certificate'] = "/etc/gitlab/ssl/bundle.crt"
				5. nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/lcsr-domain.key"
		20. Fin
	3. Upgrade Gitlab 8.10.3 to latest Omnibus install

		1. Testing one single version hop:

			1. curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
			2. sudo apt-get install gitlab-ce=8.11.11-ce.0
			3. Success!
		2. Jumping a few more versions:

			1. sudo apt-get install gitlab-ce=8.17.8-ce.0
			2. sudo apt-get install gitlab-ce=9.0.13-ce.0
			3. sudo apt-get install gitlab-ce=10.0.7-ce.0
			4. "apt-get upgrade" to upgrade to latest version 10.6.4
	4. Party!


