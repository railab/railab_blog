=============
www.railab.me
=============

This is the www.railab.me website source code.

Mechatronics engineering and other fun stuff from raiden00.

Powered by Nikola - a static page generator.

Live on http://www.railab.me
  
Theme: https://bootswatch.com/darkly

Notes
-----

Install Nikola::

  $ python3 -m venv nikola-env
  $ cd nikola-env
  $ bin/python -m pip install -U pip setuptools wheel
  $ bin/python -m pip install -U "Nikola[extras]"
  $ cd ..

Create and activate the environment::

   $ . nikola-env/bin/activate

Build::

  $ cd railab_blog
  $ nikola build

Install Isso::

  sudo apt-get install python3-dev sqlite3 build-essential
  sudo snap install docker
  docker pull ghcr.io/isso-comments/isso:latestXSXS

Add these in /etc/apache2/sites-enabled/ config::

  # Proxy for Isso commenting
  ProxyPass /isso/ http://localhost:8080
  ProxyPassReverse /isso/ http://localhost:8080
