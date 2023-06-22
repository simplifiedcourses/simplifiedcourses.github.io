git pull
JEKYLL_ENV=production jekyll build
cp -r _site/* ./var/www/html