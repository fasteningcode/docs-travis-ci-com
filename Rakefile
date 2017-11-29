#!/usr/bin/env rake

require 'html-proofer'

task default: :test

desc 'Runs the tests!'
task test: :build do
  Rake::Task['run_html_proofer'].invoke
end

desc 'Builds the site'
task build: %i[remove_output_dir regen] do
  rm_f '.jekyll-metadata'
  sh 'bundle exec jekyll build --config=_config.yml'
end

desc 'Remove the output dir'
task :remove_output_dir do
  rm_rf('_site')
end

def print_line_containing(file, str)
  File.open(file).grep(/#{str}/).each do |line| puts "#{file}: #{line}" end
end

def dns_txt(hostname)
  require 'faraday'
  require 'json'

  JSON.parse(
    Faraday.get("https://dig.jsondns.org/IN/#{hostname}/TXT").body
  ).fetch('answer').fetch(0).fetch('rdata').fetch(0).split.map(&:strip)
end

desc 'Lists files containing beta features'
task :list_beta_files do
  files = FileList.new('**/*.md')
  files.exclude("_site/*", "STYLE.md")
  for f in files do
    print_line_containing(f, '\.beta')
  end
end

desc 'Runs the html-proofer test'
task :run_html_proofer do
  # seems like the build does not render `%3*`,
  # so let's remove them for the check
  url_swap = {
    /%3A\z/ => '',
    /%3F\z/ => '',
    /-\.travis\.yml/ => '-travisyml'
  }

  tester = HTMLProofer.check_directory('./_site', {
                              :url_swap => url_swap,
                              :internal_domains => ["docs.travis-ci.com"],
                              :connecttimeout => 600,
                              :only_4xx => true,
                              :typhoeus => { :ssl_verifypeer => false, :ssl_verifyhost => 0, :followlocation => true },
                              :url_ignore => ["https://www.appfog.com/", /itunes\.apple\.com/, /coverity.com/, /articles201769485/],
                              :file_ignore => ["./_site/api/index.html", "./_site/user/languages/erlang/index.html",
                              "./_site/user/languages/objective-c/index.html",
                              "./_site/user/reference/osx/index.html"]
                            })
  tester.run
end

desc 'Runs the html-proofer test for internal links only'
task :run_html_proofer_internal do
  # seems like the build does not render `%3*`,
  # so let's remove them for the check
  url_swap = {
    /%3A\z/ => '',
    /%3F\z/ => '',
    /-\.travis\.yml/ => '-travisyml'
  }

  tester = HTMLProofer.check_directory('./_site', {
                              :url_swap => url_swap,
                              :disable_external => true,
                              :internal_domains => ["docs.travis-ci.com"],
                              :connecttimeout => 600,
                              :only_4xx => true,
                              :typhoeus => { :ssl_verifypeer => false, :ssl_verifyhost => 0, :followlocation => true },
                              :file_ignore => ["./_site/api/index.html", "./_site/user/languages/erlang/index.html",
                              "./_site/user/languages/objective-c/index.html",
                              "./_site/user/reference/osx/index.html"]
                            })
  tester.run
end

file '_data/trusty-language-mapping.json' do |t|
  source = File.join(
    'https://raw.githubusercontent.com',
    'travis-infrastructure/terraform-config/master/aws-production-2',
    'generated-language-mapping.json'
  )

  fail unless sh "curl -sSfL -o '#{t.name}' '#{source}'"
end

file '_data/trusty_language_mapping.yml' => [
  '_data/trusty-language-mapping.json'
] do |t|
  require 'json'
  require 'yaml'

  File.write(
    t.name,
    YAML.dump(JSON.load(File.read('_data/trusty-language-mapping.json')))
  )

  puts "Updated #{t.name}"
end

file '_data/ec2-public-ips.json' do |t|
  source = File.join(
    'https://raw.githubusercontent.com',
    'travis-infrastructure/terraform-config/master/aws-shared-2',
    'generated-public-ip-addresses.json'
  )

  fail unless sh "curl -sSfL -o '#{t.name}' '#{source}'"
end

file '_data/ec2_public_ips.yml' => '_data/ec2-public-ips.json' do |t|
  require 'json'
  require 'yaml'

  data = JSON.parse(File.read('_data/ec2-public-ips.json'))
  by_site = %w[com org].map do |site|
    [
      site,
      data['ips_by_host'].find { |d| d['host'] =~ /-#{site}-/ }
    ]
  end

  File.write(t.name, YAML.dump(by_site.to_h))

  puts "Updated #{t.name}"
end

file '_data/gce_ip_range.yml' do |t|
  require 'ipaddr'
  require 'yaml'

  # Using steps described in https://cloud.google.com/compute/docs/faq#where_can_i_find_short_product_name_ip_ranges
  # we populate the range of IP addresses for GCE instances
  dns_root = ENV.fetch(
    'GOOGLE_DNS_ROOT', '_cloud-netblocks.googleusercontent.com'
  )

  blocks = dns_txt(dns_root).grep(/^include:/).map do |l|
    dns_txt(l.sub(/^include:/, '')).grep(/^ip4:/)
                                   .map { |l| l.sub(/^ip4:/, '') }
  end

  File.write(
    t.name,
    YAML.dump(
      'ip_ranges' => blocks.flatten
                           .compact
                           .sort { |a, b| IPAddr.new(a) <=> IPAddr.new(b) }
    )
  )

  puts "Updated #{t.name}"
end

desc 'Refresh generated files'
task regen: [
  :clean,
  '_data/ec2_public_ips.yml',
  '_data/gce_ip_range.yml',
  '_data/trusty_language_mapping.yml',
]

desc 'Remove generated files'
task :clean do
  rm_f(%w[
    _data/ec2_public_ips.yml
    _data/ec2-public-ips.json
    _data/gce_ip_range.yml
    _data/trusty-language-mapping.json
    _data/trusty_language_mapping.yml
  ])
end

desc 'Start Jekyll server'
task serve: :regen do
  sh "bundle exec jekyll serve --config=_config.yml"
end

namespace :assets do
  task precompile: :build
end
