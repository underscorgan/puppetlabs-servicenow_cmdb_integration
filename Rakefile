# frozen_string_literal: true

require 'puppet_litmus/rake_tasks' if Bundler.rubygems.find_name('puppet_litmus').any?
require 'puppetlabs_spec_helper/rake_tasks'
require 'puppet-syntax/tasks/puppet-syntax'
require 'puppet_blacksmith/rake_tasks' if Bundler.rubygems.find_name('puppet-blacksmith').any?
require 'github_changelog_generator/task' if Bundler.rubygems.find_name('github_changelog_generator').any?
require 'puppet-strings/tasks' if Bundler.rubygems.find_name('puppet-strings').any?

def changelog_user
  return unless Rake.application.top_level_tasks.include? "changelog"
  returnVal = nil || JSON.load(File.read('metadata.json'))['author']
  raise "unable to find the changelog_user in .sync.yml, or the author in metadata.json" if returnVal.nil?
  puts "GitHubChangelogGenerator user:#{returnVal}"
  returnVal
end

def changelog_project
  return unless Rake.application.top_level_tasks.include? "changelog"

  returnVal = nil
  returnVal ||= begin
    metadata_source = JSON.load(File.read('metadata.json'))['source']
    metadata_source_match = metadata_source && metadata_source.match(%r{.*\/([^\/]*?)(?:\.git)?\Z})

    metadata_source_match && metadata_source_match[1]
  end

  raise "unable to find the changelog_project in .sync.yml or calculate it from the source in metadata.json" if returnVal.nil?

  puts "GitHubChangelogGenerator project:#{returnVal}"
  returnVal
end

def changelog_future_release
  return unless Rake.application.top_level_tasks.include? "changelog"
  returnVal = "v%s" % JSON.load(File.read('metadata.json'))['version']
  raise "unable to find the future_release (version) in metadata.json" if returnVal.nil?
  puts "GitHubChangelogGenerator future_release:#{returnVal}"
  returnVal
end

PuppetLint.configuration.send('disable_relative')

if Bundler.rubygems.find_name('github_changelog_generator').any?
  GitHubChangelogGenerator::RakeTask.new :changelog do |config|
    raise "Set CHANGELOG_GITHUB_TOKEN environment variable eg 'export CHANGELOG_GITHUB_TOKEN=valid_token_here'" if Rake.application.top_level_tasks.include? "changelog" and ENV['CHANGELOG_GITHUB_TOKEN'].nil?
    config.user = "#{changelog_user}"
    config.project = "#{changelog_project}"
    config.future_release = "#{changelog_future_release}"
    config.exclude_labels = ['maintenance']
    config.header = "# Change log\n\nAll notable changes to this project will be documented in this file. The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](http://semver.org)."
    config.add_pr_wo_labels = true
    config.issues = false
    config.merge_prefix = "### UNCATEGORIZED PRS; GO LABEL THEM"
    config.configure_sections = {
      "Changed" => {
        "prefix" => "### Changed",
        "labels" => ["backwards-incompatible"],
      },
      "Added" => {
        "prefix" => "### Added",
        "labels" => ["feature", "enhancement"],
      },
      "Fixed" => {
        "prefix" => "### Fixed",
        "labels" => ["bugfix"],
      },
    }
  end
else
  desc 'Generate a Changelog from GitHub'
  task :changelog do
    raise <<EOM
The changelog tasks depends on unreleased features of the github_changelog_generator gem.
Please manually add it to your .sync.yml for now, and run `pdk update`:
---
Gemfile:
  optional:
    ':development':
      - gem: 'github_changelog_generator'
        git: 'https://github.com/skywinder/github-changelog-generator'
        ref: '20ee04ba1234e9e83eb2ffb5056e23d641c7a018'
        condition: "Gem::Version.new(RUBY_VERSION.dup) >= Gem::Version.new('2.2.2')"
EOM
  end
end

# ACCEPTANCE TEST RAKE TASKS + HELPERS

namespace :acceptance do
  require 'puppet_litmus/rake_tasks'
  require_relative './spec/support/acceptance/helpers'
  include TargetHelpers

  desc 'Provisions the VMs. This is currently just the master'
  task :provision_vms do
    if File.exist?('inventory.yaml')
      # Check if a master VM's already been setup
      begin
        uri = master.uri
        puts("A master VM at '#{uri}' has already been set up")
        next
      rescue TargetNotFoundError
        # Pass-thru, this means that we haven't set up the master VM
      end
    end

    provision_list = ENV['PROVISION_LIST'] || 'acceptance'
    Rake::Task['litmus:provision_list'].invoke(provision_list)
  end

  # TODO: This should be refactored to use the https://github.com/puppetlabs/puppetlabs-peadm
  # module for PE setup
  desc 'Sets up PE on the master'
  task :setup_pe do
    master.bolt_run_script('spec/support/acceptance/install_pe.sh')
  end

  desc 'Sets up the ServiceNow instance'
  task :setup_servicenow_instance do
    # Start the mock ServiceNow instance. Note that the script's already idempotent
    # (and fast enough) so we don't need to check the inventory file for the created
    # ServiceNow instance
    puts("Starting the mock ServiceNow instance at the master (#{master.uri})")
    master.bolt_upload_file('./spec/support/acceptance/servicenow', '/tmp/servicenow')
    master.bolt_run_script('spec/support/acceptance/start_mock_servicenow_instance.sh')

    # Update the inventory file
    puts('Updating the inventory.yaml file with the mock ServiceNow instance credentials')
    inventory_hash = LitmusHelpers.inventory_hash_from_inventory_file
    servicenow_group = inventory_hash['groups'].find { |g| g['name'] =~ %r{servicenow} }
    unless servicenow_group
      servicenow_group = { 'name' => 'servicenow_nodes' }
      inventory_hash['groups'].push(servicenow_group)
    end
    servicenow_group['targets'] = [{
      'uri' => "#{master.uri}:1080",
      'config' => {
        'transport' => 'remote',
        'remote' => {
          'user' => 'mock_user',
          'password' => 'mock_password'
        }
      },
      'vars' => {
        'roles' => ['servicenow_instance'],
      }
    }]
    write_to_inventory_file(inventory_hash, 'inventory.yaml')
  end

  desc 'Sets up the ServiceNow CMDB with entries for each VM'
  task :setup_servicenow_cmdb do
    record = CMDBHelpers.get_target_record(master)
    unless record.nil?
      puts("A CMDB record's already been created for the master (sys_id = #{record['sys_id']})")
      next
    end

    # CMDB record doesn't exist so create it
    puts("Creating the master's CMDB record ...")
    fields_template = JSON.parse(File.read('spec/support/acceptance/cmdb_record_template.json'))
    fields = CMDBHelpers.create_record(fields_template.merge('fqdn' => master.uri))
    puts("Created the CMDB record (sys_id = #{fields['sys_id']})")
  end

  desc 'Installs the module on the master'
  task :install_module do
    Rake::Task['litmus:install_module'].invoke(master.uri)
  end

  desc 'Set up the test infrastructure'
  task :setup do
    tasks = [
      :provision_vms,
      :setup_pe,
      :setup_servicenow_instance,
      :setup_servicenow_cmdb,
      :install_module,
    ]

    tasks.each do |task|
      task = "acceptance:#{task}"
      puts("Invoking #{task}")
      Rake::Task[task].invoke
      puts("")
    end
  end

  desc 'Runs the tests'
  task :run_tests do
    puts("Running the tests ...\n")
    unless system('bundle exec rspec ./spec/acceptance')
      # system returned false which means rspec failed. So exit 1 here
      exit 1
    end
  end

  desc 'Teardown the ServiceNow CMDB'
  task :tear_down_servicenow_cmdb do
    CMDBHelpers.delete_target_record(master)
  end

  desc 'Teardown the setup'
  task :tear_down do
    puts("Tearing down the test infrastructure ...\n")

    # Teardown the test CMDB
    begin
      using_mock_instance = servicenow_instance.uri =~ Regexp.new(Regexp.escape(master.uri))
      unless using_mock_instance
        Rake::Task['acceptance:tear_down_servicenow_cmdb'].invoke
      end
    rescue TargetNotFoundError
      # Pass-thru, this means that the tear_down task was called before
      # the ServiceNow instance was setup
    end

    # Teardown the master
    Rake::Task['litmus:tear_down'].invoke(master.uri)

    # Delete the inventory file
    FileUtils.rm_f('inventory.yaml')
  end

  desc 'Task for CI'
  task :ci_run_tests do
    begin
      Rake::Task['acceptance:setup'].invoke
      Rake::Task['acceptance:run_tests'].invoke 
    ensure
      Rake::Task['acceptance:tear_down'].invoke
    end
  end
end
