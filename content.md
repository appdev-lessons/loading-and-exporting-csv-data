# Loading and Exporting CSV Data

## Loading data from a CSV file into your database

### 1. Setup

First, create a folder inside of the `lib` folder called `csvs`

Put your CSV file into the `lib/csvs` folder. In the example below, the file is called `real_estate_transactions.csv`

You should already have an ActiveRecord model ready to go to store the data from the CSV; in our example below, we have one called `Transaction`. The column names in your model don't have to match up exactly with the headings in the CSV.

Let's suppose we ultimately want to run a command named `rake slurp:transactions` which will slurp the data from `real_estate_transactions.csv` and put it in our database.

First, let's create a custom rake task with this name:

```
rails generate task slurp transactions
```

This will create a file called `lib/tasks/slurp.rake` that looks like this:

```ruby
namespace :slurp do
  desc "TODO"
  task transactions: :environment do
  end

end
```

You can now add the code described in the rest of this guide between the lines `task transactions: :environment do` and the first `end`:

### 2. Read in a CSV file

```ruby
require "csv"

csv_text = File.read(Rails.root.join("lib", "csvs", "real_estate_transactions.csv"))
puts csv_text
```

The first line requires the Ruby CSV library we need to properly parse the CSV data. The next line reads in the CSV file into a variable. The last line prints the contents of the variable. When you run `rake slurp:transactions` you should see a wall of text representing your CSV data. It's a first step, but we've still got a lot of work to do.

We'll keep building off this code until we've created a working seeds file. You should be able to run `rake slurp:transactions` at the end of each step

### 3. Parse the CSV

```ruby
require "csv"

csv_text = File.read(Rails.root.join("lib", "csvs", "real_estate_transactions.csv"))
csv = CSV.parse(csv_text, :headers => true, :encoding => "ISO-8859-1")
puts csv
```

The new line converts the CSV file into a structure that Ruby can read. The `:headers => true` option tells the parser that the first line in the file has column headings in it, not a row of data.

### 4. Looping through the parsed data

```ruby
require "csv"

csv_text = File.read(Rails.root.join("lib", "csvs", "real_estate_transactions.csv"))
csv = CSV.parse(csv_text, :headers => true, :encoding => "ISO-8859-1")
csv.each do |row|
  puts row.to_hash
end
```

This new addition loops through the entire CSV file and converts each row of the document into a hash. The headers of the CSV file will be used as keys for the hash because we added the `:headers => true` option in our previous step.

### 5. Create a database object from each row

```ruby
require "csv"

csv_text = File.read(Rails.root.join("lib", "csvs", "real_estate_transactions.csv"))
csv = CSV.parse(csv_text, :headers => true, :encoding => "ISO-8859-1")
csv.each do |row|
  t = Transaction.new
  t.street_address = row["street"]
  t.city = row["city"]
  t.zip = row["zip"]
  t.state = row["state"]
  t.bedrooms = row["beds"]
  t.square_feet = row["sq_feet"]
  t.category = row["type"]
  t.sold_on = row["sale_date"]
  t.price = row["price"]
  t.lat = row["latitude"]
  t.lng = row["longitude"]
  t.save
  puts "#{t.street_address}, #{t.zip} saved"
end

puts "There are now #{Transaction.count} rows in the transactions table"
```

## Exporting Data as CSV

### Define a Class method

For example, if you wanted to export Photos records from Photogram.

In the `photo.rb` model, define a class method called `to_csv`.

```ruby
class Photo < ApplicationRecord

  # ...
  def self.to_csv
    # ...
  end
end
```

#### Choose the headers

You need to choose what headers you want to appear in the generated CSV. Create a variable called `headers` that contains an Array with each of the headers that you want to appear in the generated CSV.

```ruby
class Photo < ApplicationRecord

  # ...
  def self.to_csv
    photos = self.all 
    headers = ["id", "caption", "image", "owner_id"]
    csv = CSV.generate(headers: true) do |csv|
      csv << headers
      # ...
    end
    return csv
  end
end
```

`self.all` is similar to `Photo.all`. `photos` determines which `Photo` records will enter the exported CSV.

#### Choose the value for each row in the CSV

For the other rows besides the header, add a loop over your list of records (`photos`, in this case).

Inside the loop, create an Array that represents all the data that will appear in each row. The value and order of the elements in this `row` Array should match the order of values in the headers.

```ruby
class Photo < ApplicationRecord

  # ...
  def self.to_csv
    photos = self.all
    headers = ["id", "caption", "image", "owner_id"]
    csv = CSV.generate(headers: true) do |csv|
      csv << headers
      photos.each do |photo|
        row = []
        row.push(photo.id)
        row.push(photo.caption)
        row.push(photo.image)
        row.push(photo.owner_id)
        csv << row
      end
    end
    return csv
  end
end
```

### Create a route to download the file

```ruby
# config/routes.rb
get("/export_photos", { :controller => "photos", :action => "export" })

# app/controllers/photos_controller.rb
class PhotosController < ApplicationController
  # ...
  def export
    photos = Photo.all
    respond_to do |format|
      format.csv do 
        send_data(photos.to_csv, { :filename => "my_photos.csv"})
      end
    end
  end
  # ...
end
```

### Add download link to View template

```erb
<a href="/export_photos.csv">Export as CSV</a>
```

Now clicking on "Export as CSV" should open the download file dialog with the generated CSV.

---
