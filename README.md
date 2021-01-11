# My Blog

My blog is based on [Pelican](https://blog.getpelican.com).

```sh
$ git clone https://github.com/marselester/marselester.github.com.git
$ cd ./marselester.github.com/
$ git clone --recursive https://github.com/getpelican/pelican-themes
$ git clone --recursive https://github.com/getpelican/pelican-plugins
$ virtualenv venv
$ . venv/bin/activate
(venv) $ pip install -r requirements.txt
(venv) $ make generate
(venv) $ open index.html
```
