require "bundler"
begin
  Bundler.setup
  Bundler::GemHelper.install_tasks
rescue
  raise "You need to install a bundle first. Try 'thor version:use 3.2.13'"
end

require 'yaml'
require 'rspec'
require 'rspec/core/rake_task'
require 'cucumber/rake/task'

def rails_template_command
  require "rails/version"
  "app:template"
end

desc "Run all examples"
RSpec::Core::RakeTask.new(:spec) do |t|
  t.ruby_opts = %w[-w]
end

Cucumber::Rake::Task.new(:cucumber)

namespace :generate do
  desc "generate a fresh app with rspec installed"
  task :app do |_task|
    unless File.directory?('./tmp/example_app')
      bindir = File.expand_path("bin")

      # Rails 4+ cannot use a `rails` binstub generated by Bundler
      sh "rm -f #{bindir}/rails"
      sh "bundle exec rails new ./tmp/example_app --no-rc --skip-javascript --skip-bootsnap --skip-sprockets " \
         "--skip-git --skip-test-unit --skip-listen --skip-bundle --skip-spring " \
         "--template=example_app_generator/generate_app.rb"

      in_example_app do
        sh "./ci_retry_bundle_install.sh 2>&1"
        # Rails 4+ cannot use a `rails` binstub generated by Bundler
        sh "bundle binstubs bundler rspec-core rake --force"
        sh "bundle binstubs railties" unless File.exist?("bin/rails")

        application_file = File.read("config/application.rb")
        application_file.gsub!("config.assets.enabled = true", "config.assets.enabled = false")
        application_file.gsub!('# require "sprockets/railtie"', 'require "sprockets/railtie"')

        sh "rm config/application.rb"

        File.open("config/application.rb", "w") do |f|
          f.write(application_file)
        end
      end
    end
  end

  desc "generate a bunch of stuff with generators"
  task :stuff do
    in_example_app "bin/rake #{rails_template_command} LOCATION='../../example_app_generator/generate_stuff.rb'"
  end
end

def in_example_app(*command_opts)
  app_dir = './tmp/example_app'
  if Hash === command_opts.last
    opts = command_opts.pop
    app_dir = opts.fetch(:app_dir, app_dir)
  end
  Dir.chdir(app_dir) do
    Bundler.with_original_env do
      ENV["BUNDLE_GEMFILE"] = "Gemfile"
      sh *command_opts unless command_opts.empty?
      yield if block_given?
    end
  end
end

namespace :db do
  task :migrate do
    in_example_app "bin/rake db:migrate"
  end

  namespace :test do
    task :prepare do
      in_example_app "bin/rake db:test:prepare"
    end
  end
end

desc "run a variety of specs against the generated app"
task :smoke do
  in_example_app "LOCATION='../../example_app_generator/run_specs.rb' bin/rake #{rails_template_command} --backtrace"
end

namespace :smoke do
  desc "create a new example app with generated specs and run them"
  task app: ["clobber:app", "generate:app", "generate:stuff", :smoke]

  desc "run in the example app"
  task :run, [:cmd] do |_t, args|
    unless ENV['SKIP_GENERATE']
      ["clobber:app", "generate:app", "generate:stuff"].map { |task| Rake::Task[task].invoke }
    end

    in_example_app args.cmd.to_s
  end

  desc "run RSPEC_OPTS environment variable in the example app for local dev"
  task :rspec do
    in_example_app "LOCATION='../../example_app_generator/run_specs.rb' bin/rspec #{ENV.fetch("RSPEC_OPTS")}"
  end
end

desc 'clobber generated files'
task :clobber do
  rm_rf "pkg"
  rm_rf "tmp"
  rm_rf "doc"
  rm_rf ".yardoc"
end

namespace :clobber do
  desc "clobber the generated app"
  task :app do
    rm_rf "tmp/example_app"
  end
end

namespace :no_active_record do
  example_app_dir = './tmp/no_ar_example_app'

  desc "run a variety of specs against a non-ActiveRecord generated app"
  task :smoke do
    in_example_app "LOCATION='../../example_app_generator/run_specs.rb' bin/rake #{rails_template_command} --backtrace",
                   app_dir: example_app_dir
  end

  namespace :smoke do
    desc "create a new example app without active record including generated specs and run them"
    task app: [
      "no_active_record:clobber",
      "no_active_record:generate:app",
      "no_active_record:generate:stuff",
      "no_active_record:smoke",
    ]

    desc "run RSPEC_OPTS environment variable in the example app for local dev"
    task :rspec do
      in_example_app "LOCATION='../../example_app_generator/run_specs.rb' bin/rspec #{ENV.fetch("RSPEC_OPTS")}", app_dir: example_app_dir
    end
  end

  desc "remove the old non-ActiveRecord app"
  task :clobber do
    rm_rf example_app_dir
  end

  namespace :generate do
    desc "generate a fresh app with no active record"
    task :app do
      unless File.directory?(example_app_dir)
        bindir = File.expand_path("bin")

        # Rails 4+ cannot use a `rails` binstub generated by Bundler
        sh "rm -f #{bindir}/rails"
        sh "bundle exec rails new #{example_app_dir} --no-rc --skip-active-record --skip-javascript --skip-bootsnap " \
           "--skip-sprockets --skip-git --skip-test-unit --skip-listen --skip-bundle --skip-spring " \
           "--skip-action-text --template=example_app_generator/generate_app.rb"

        in_example_app(app_dir: example_app_dir) do
          sh "./ci_retry_bundle_install.sh 2>&1"
          # Rails 4+ cannot use a `rails` binstub generated by Bundler
          sh "bundle binstubs bundler rspec-core rake --force"
          sh "bundle binstubs railties" unless File.exist?("bin/rails")

          application_file = File.read("config/application.rb")
          application_file.gsub!("config.assets.enabled = true", "config.assets.enabled = false")
          application_file.gsub!('# require "sprockets/railtie"', 'require "sprockets/railtie"')

          sh "rm config/application.rb"

          File.open("config/application.rb", "w") do |f|
            f.write(application_file)
          end
        end
      end
    end

    desc "generate a bunch of stuff with generators"
    task :stuff do
      in_example_app "bin/rake #{rails_template_command} LOCATION='../../example_app_generator/generate_stuff.rb'", app_dir: example_app_dir
    end
  end
end

task acceptance: ['smoke:app', 'no_active_record:smoke:app', :cucumber]

task default: [:spec, :acceptance]

task :verify_private_key_present do
  private_key = File.expand_path('~/.gem/rspec-gem-private_key.pem')
  unless File.exist?(private_key)
    raise "Your private key is not present. This gem should not be built without it."
  end
end

task build: :verify_private_key_present

namespace :coverage do
  desc "Collates all result sets generated by the different test runners"
  task :report do
    require 'simplecov'
    require 'simplecov_json_formatter'

    SimpleCov.collate Dir["/tmp/aggregated_results/resultset*"] do
      formatter SimpleCov::Formatter::MultiFormatter.new([
        SimpleCov::Formatter::HTMLFormatter,
        SimpleCov::Formatter::JSONFormatter
      ])
    end
  end
end
