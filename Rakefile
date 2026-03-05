require 'rake/testtask'

task :default => :test

Rake::TestTask.new do |t|
  t.libs << "test"
  t.test_files = FileList["test/*_test.rb"]
  t.warning = true
end

task :gems do
  Dir["*.gemspec"].each do |gemspec|
    system "gem build #{gemspec}"
  end
end

desc "Download latest PhoneNumberMetadata.xml and generate JSON database into tmp/"
task :generate do
  require "tempfile"
  require "fileutils"

  source_url = "https://raw.githubusercontent.com/google/libphonenumber/master/resources/PhoneNumberMetadata.xml"
  output_dir = File.expand_path("tmp", __dir__)
  output_path = File.join(output_dir, "global_phone.json")

  FileUtils.mkdir_p(output_dir)

  puts "Source:  #{source_url}"
  puts "Output:  #{output_path}"
  puts

  xml_file = Tempfile.new("PhoneNumberMetadata.xml")
  begin
    puts "Downloading metadata XML..."
    unless system("curl", "-fsSL", "-o", xml_file.path, source_url)
      abort "ERROR: failed to download #{source_url}"
    end
    puts "Downloaded #{File.size(xml_file.path)} bytes"
    puts

    puts "Generating JSON database..."
    unless system("ruby", "-Ilib", "bin/global_phone_dbgen", xml_file.path, out: output_path)
      abort "ERROR: global_phone_dbgen failed"
    end
  ensure
    xml_file.close
    xml_file.unlink
  end

  puts "\nGenerated #{output_path} (#{File.size(output_path)} bytes)"
end
