---
layout: post
title: Using Miniexiftool and Paperclip to plot image locations
---

Have you ever wondered to yourself how the Photos app on your iPhone always knows where you took that picture? What most people don't realize is that the information of where that picture was taken (among many other things) is often stored within the metadata of that photo.

Using the power of a few gems and some good ol' ruby code, you can harness the power of that metadata and do fun (or creepy) things in your applications. Let's create an application that takes user photos and plots them on a map.

First off, we'll need to install some software on your machine that allows you to read image metadata.

```bash
brew install exiftool
```

> `exiftool` is the Perl application that allows you to read the metadata, but don't be thrown off by this requirement. While installing this application with brew may not seem ideal for portability to services like Heroku, there is a [gem](https://github.com/wilg/mini_exiftool_vendored) that includes this Perl library and will allow you to run this in the herokusphere.

Great. Let's create a new Rails application:

```bash
rails new creeper
cd creeper
```

And create a `Picture` model that Paperclip will later utilize to upload files:

```bash
rails g scaffold picture title:string
rake db:migrate
```

Next up, edit your `Gemfile` and add the following:

```ruby
gem 'paperclip'
gem 'mini_exiftool'
```

> Again, if you plan on going to Heroku, you should probably be using `gem 'mini_exiftool_vendored'` instead, which includes the Perl Exiftool library.

And run `bundle` to install those gems.

### Configuring Paperclip

Now let's configure Paperclip. These instructions come right from their [readme](https://github.com/thoughtbot/paperclip/blob/master/README.md).

1. Edit your `Picture` model (`app/models/picture.rb`) and add:

    ```ruby
    has_attached_file :image
    validates_attachment_content_type :image, content_type: /\Aimage\/.*\Z/
    ```

2. Create a migration to add the Paperclip attributes to the `Picture` model:

    ```bash
    rails g migration AddImageToPictures
    ```

3. In the newly generated `/db/migrate/` file, replace the blank `change` method with the following code:

    ```ruby
    def self.up
      add_attachment :pictures, :image
    end

    def self.down
      remove_attachment :pictures, :image
    end
    ```

4. And then migrate the database:

    ```bash
    rake db:migrate
    ```

5. Great. Now the model is set up, but we need to make sure that our views have a way of uploading pictures. Let's update `app/views/pictures/_form.html.erb` and add the following immediately before the submit button's `div`:

    ```erb
    <div class="field">
      <%= f.label :image %><br>
      <%= f.file_field :image %>
    </div>
    ```

6. Also, let's make sure that strong_params allows us to add an image. In `app/controllers/pictures_controller.rb`, let's add `:image` to the `.permit()`:

    ```ruby
    params.require(:image).permit(:title, :image)
    ```

7. Sweet! We can now add pictures. The last step is to make sure that we can see the images once we've added them. In `app/views/pictures/show.html.erb`, let's add a way to view our images before the `<%= link_to 'Edit' %>`:

    ```erb
    <%= image_tag @picture.image.url, width: 600 %><br>
    ```

### Configuring Geolocation

Perfect. Paperclip is installed and should be working locally. Next up we need to create two attributes (`latitude` and `longitude`) on the `Picture` model.

```bash
rails g migration AddLatitudeAndLongitudeToPicture latitude:float longitude:float
rake db:migrate
```

### The Good Stuff

Awesome. Now we have a way to upload images, the next step is to make sure that we're copying the exif data to the model when Paperclip saves an image. Luckily, Paperclip gives us a method called `after_post_process` that we can call in our model that will direct Paperclip to run the method passed as an argument right after it finishes the post processing of the image. Let's take advantage of that in `app/models/picture.rb`:

```ruby
after_post_process :copy_exif_data
```

Once we add that, let's add the method `copy_exif_data` as well as a helper method that will allow us to parse latitude and longitudes from the exif data:

```ruby
private

def copy_exif_data
  exif_data = MiniExiftool.new(image.queued_for_write[:original].path)
  self.latitude = parse_latlong(exif_data['gpslatitude'])
  self.longitude = parse_latlong(exif_data['gpslongitude'])
end

def parse_latlong(latlong)
  latlong.scan(/(.*) deg (.*)' (.*)" (.*)/).map do |d,m,s,r|
    calc = d.to_f + m.to_f/60 + s.to_f/3600
    if ['S', 'W'].include? r
      -calc
    else
      calc
    end
  end.last
end
```

### Google Maps

Sweet, so we have a creepy application set up on the back end, let's show the user where the image was taken on the front end. We're going to be using an iframe to embed a google map on the `Picture#show` page. In order to do that, we have to get an API key from Google. We can get one, here: [https://code.google.com/apis/console](https://code.google.com/apis/console).

This will allow us to query Google Maps, but we want to make sure that if this is an open source repository, we're never adding that API key to version control, since if it got into the wrong hands, our Google Maps functionality could be disabled by Google.

Enter `dotenv-rails`. Dotenv is a way for us to store sensitive information in a file (`.env`) and not have it checked into version control.

First, let's add `dotenv-rails` to our Gemfile. We _have_ to make sure that this is done at the top of the `Gemfile`, right after `source`:

```ruby
gem 'dotenv-rails', groups: [:development, :test]
```

And bundle to install the gem:

```bash
bundle
```

And then we add our API key to `.env` (obviously replace with your own, the below one is not functional):

```
GOOGLE_MAPS_API_KEY=AIzaSyAfCI6RpkFNMbrnUDlIV4MAAbaRGiU-2k
```

This will now allow us to access the key as so anywhere in our Rails code: `ENV['GOOGLE_MAPS_API_KEY']`. Which is perfect, because we'll need that in our view to display the map. Let's edit `app/views/pictures/show.html.erb` and put the following code after the `image_tag`:

```erb
<iframe
    width="600"
    height="450"
    frameborder="0" style="border:0"
    src="https://www.google.com/maps/embed/v1/place?key=<%= ENV['GOOGLE_MAPS_API_KEY'] %>
    &q=<%= "#{@picture.latitude},#{@picture.longitude}" %>">
</iframe>
<br>
```

Let's fire up our rails server with `rails s`, and now we should be displaying the location of where our image was taken! Go ahead and upload a new image and see if it can accurately pinpoint where that image was taken.

![creeper_screenshot](http://i.imgur.com/udAXXCx.png)
