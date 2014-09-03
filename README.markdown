#  Dokku Cert Install Plugin

This plugin lets you inject custom public key certificates into your [Dokku](https://github.com/progrium/dokku) app before the build step. You can use this with a caching HTTPS proxy in your build step, for example.

## Usage

On your Dokku server, create a `certs` directory in the application's home directory:

    dokku$ mkdir /home/dokku/my-app/certs

Then place any certificates you want installed in your container into that directory, making sure they end with `.crt`. Then you can install this plugin into Dokku:

    $ cd /var/lib/dokku/plugins
    $ sudo git clone https://github.com/GeoCENS/dokku-cert-install.git cert-install
    $ sudo dokku plugins-install

Then deploy or [rebuild](https://github.com/scottatron/dokku-rebuild) your Dokku application.

## Example

You can use this with [package-proxy](https://github.com/lox/package-proxy) to add local caching for RubyGems or Npm. This can have a noticeable improvement in deploy times.

You will also need the [dokku-user-env-compile](https://github.com/GeoCENS/dokku-user-env-compile) plugin, although [dokku-build-env](https://github.com/cameron-martin/dokku-build-env) should also work with minor changes.

Start by setting up package-proxy with Docker:

    $ sudo docker run --tty --sig-proxy=false --interactive --rm --publish 3142:3142 --name package-proxy lox24/package-proxy:latest

Use Ctrl-P, Ctrl-Q, then Ctrl-C to detach once it is running. Get the IP of the host running the package-proxy container, or the container's IP if it is running on the same Docker host as Dokku.

Next get a copy of the package-proxy public key:

    $ sudo docker cp package-proxy:certs/packageproxy-ca.crt .

Copy this key to your Dokku app's certs directory:

    $ sudo mkdir /home/dokku/my-ruby-app/certs
    $ sudo cp packageproxy-ca.crt /home/dokku/my-ruby-app/certs/.

And add the proxy information to your ENV file:

    $ sudo cat >> /home/dokku/my-ruby-app/ENV <<EOF
    export http_proxy=http://10.0.1.1:3142/
    export https_proxy=https://10.0.1.1:3142/
    EOF

Then install the plugins:

    $ cd /var/lib/dokku/plugins
    $ sudo git clone https://github.com/GeoCENS/dokku-cert-install.git cert-install
    $ sudo git clone https://github.com/GeoCENS/dokku-user-env-compile.git user-env-compile
    $ sudo dokku plugins-install

Then deploy or rebuild your Dokku application. The first run won't see much of a benefit as the cache is still cold. But you can take a look at the logs for the package-proxy container and see the cache misses. The next deployment will see a modest improvement in Ruby Gem install times, I personally had a 63s bundle step drop to a 47s build step.

As package-proxy also works with APT, Npm, and Composer, any Dokku buildpacks that respect `$http_proxy` and `$https_proxy` ENV variables and use those package managers should also see decreased deploy times.

## License

MIT
