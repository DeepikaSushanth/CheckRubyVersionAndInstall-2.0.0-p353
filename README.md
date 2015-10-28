# CheckRubyVersionAndInstall-2.0.0-p353
Playbook that checks and installs ruby version 2.0.0-p353 and removes any other version of ruby.

 ---
          - hosts: development
            remote_user: myusername
            sudo: yes
            vars:
              username: myusername
              ruby_version: 2.0.0-p353
              rbenv_home: /home/myusername/.rbenv
              rbenv_env: "RBENV_ROOT={{ rbenv_home }} PATH={{ rbenv_home }}/shims:{{ rbenv_home }}/bin:$PATH"
              rbenv_cmd: "{{ rbenv_env }} rbenv"
              mysql_root_password: password
            tasks:
              - name: pycurl (apt_repository requirement)
                apt: pkg=python-pycurl state=present
              - name: psycopg2 (postgresql_user requirement)
                apt: pkg=python-psycopg2 state=present
              - name: Add emacs repository
                apt_repository: repo='ppa:cassou/emacs' state=present
              - name: Add postgres repository
                apt_repository: repo='deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main' state=present
              - name: Update cache
                apt: update_cache=yes
              - name: Basic packages
                apt: "pkg={{ item }} state=present"
                with_items:
                  - curl
                  - git-core
                  - unzip
                  - libxslt-dev
                  - libxml2-dev
              - include: tasks/setup-nginx.yml
              - name: Packages required by rbenv
                apt: "pkg={{ item }} state=present"
                with_items:
                  - make
                  - build-essential
                  - autoconf
                  - libssl-dev
                  - libyaml-dev
                  - libreadline6
                  - libreadline6-dev
                  - zlib1g
                  - zlib1g-dev
              - name: Checkout rbenv
                git: repo=https://github.com/sstephenson/rbenv.git dest=/home/myusername/.rbenv
                sudo: no
              - name: Set rbenv environment
                template: src=templates/rbenv.sh dest=/etc/profile.d/rbenv.sh
              - name: Checkout ruby-build
                git: repo=https://github.com/sstephenson/ruby-build.git dest=/home/myusername/.rbenv/plugins/ruby-build
                sudo: no
              - name: "check ruby {{ ruby_version }} installed"
                shell: "/home/myusername/.rbenv/bin/rbenv versions | grep {{ ruby_version }}"
                register: ruby_installed
                ignore_errors: yes
                sudo: no
              - name: "install ruby {{ ruby_version }}"
                shell: "/home/myusername/.rbenv/bin/rbenv install {{ ruby_version}} creates=/home/myusername/.rbenv/versions/{{ ruby_version }}"
                when: ruby_installed|failed
                sudo: no
              - name: Rbenv rehash
                shell: /home/myusername/.rbenv/bin/rbenv rehash
              - name: Set global ruby
                shell: "/home/myusername/.rbenv/bin/rbenv global {{ ruby_version }}"
                sudo: no
              - name: Postgres
                apt: pkg=postgresql-9.3 state=present force=yes
              - name: Postgres libpq-dev
                apt: pkg=libpq-dev state=present force=yes
              - name: Postgres contrib
                apt: pkg=postgresql-contrib state=present force=yes
              - name: "{{ username }} postgres user"
                sudo_user: postgres
                postgresql_user: "name={{ username }} password=password role_attr_flags=SUPERUSER"
              - name: rails postgres user
                sudo_user: postgres
                postgresql_user: name=rails password=password role_attr_flags=SUPERUSER
                # creating hstore extension requires superuser role
              - include: tasks/setup-mysql.yml
              - name: Emacs
                apt: "pkg={{ item }} state=present"
                with_items:
                  - emacs-snapshot-el
                  - emacs-snapshot
                  
              - name: Emacs configs
                git: repo=https://github.com/myusername/emacs-cask-org.git dest=/home/myusername/.emacs.d
                sudo: no
              - name: Create App
                file: "path=/home/{{ username }}/Apps state=directory"
                sudo: no
                
              - name: Download Cask
                command: "curl -fsSkL https://raw.github.com/cask/cask/master/go -o /home/{{ username }}/Apps/cask.py creates=/home/{{ username }}/.cask"
                sudo: no
                
              - name: Install Cask
                command: "python /home/{{ username }}/Apps/cask.py chdir=/home/{{ username }}/.emacs.d/ creates=/home/{{ username }}/.cask"
                sudo: no
          
              - name: cask install
                command: "/home/{{ username }}/.cask/bin/cask install chdir=/home/{{ username }}/.emacs.d/"
                sudo: no
          
          
              - name: Other required packages
                apt: "pkg={{ item }} state=present"
                with_items:
                  - imagemagick
          
              - name: apt upgrade
                apt: upgrade=safe
          
            handlers:
              - include: handlers/handlers.yml
          
