require "bundler"
Bundler::GemHelper.install_tasks
require "bundler/setup"

MOCHA_DOCS_HOST = ENV['MOCHA_DOCS_HOST'] || 'gofreerange.com'

require 'rake/testtask'

desc "Run all tests"
task 'default' => ['test', 'test:performance']

desc "Run tests"
task 'test' do
  if test_library = ENV['MOCHA_RUN_INTEGRATION_TESTS']
    Rake::Task["test:integration:#{test_library}"].invoke
  else
    Rake::Task['test:units'].invoke
    Rake::Task['test:acceptance'].invoke
  end
end

namespace 'test' do

  unit_tests = FileList['test/unit/**/*_test.rb']
  all_acceptance_tests = FileList['test/acceptance/*_test.rb']
  ruby186_incompatible_acceptance_tests = FileList['test/acceptance/stub_class_method_defined_on_*_test.rb'] + FileList['test/acceptance/stub_instance_method_defined_on_*_test.rb']
  ruby186_compatible_acceptance_tests = all_acceptance_tests - ruby186_incompatible_acceptance_tests

  desc "Run unit tests"
  Rake::TestTask.new('units') do |t|
    t.libs << 'test'
    t.test_files = unit_tests
    t.verbose = true
    t.warning = true
  end

  desc "Run acceptance tests"
  Rake::TestTask.new('acceptance') do |t|
    t.libs << 'test'
    if defined?(RUBY_VERSION) && (RUBY_VERSION >= "1.8.7")
      t.test_files = all_acceptance_tests
    else
      t.test_files = ruby186_compatible_acceptance_tests
    end
    t.verbose = true
    t.warning = true
  end

  namespace 'integration' do
    desc "Run MiniTest integration tests (intended to be run in its own process)"
    Rake::TestTask.new('minitest') do |t|
      t.libs << 'test'
      t.test_files = FileList['test/integration/mini_test_test.rb']
      t.verbose = true
      t.warning = true
    end

    desc "Run Test::Unit integration tests (intended to be run in its own process)"
    Rake::TestTask.new('test-unit') do |t|
      t.libs << 'test'
      t.test_files = FileList['test/integration/test_unit_test.rb']
      t.verbose = true
      t.warning = true
    end
  end

  # require 'rcov/rcovtask'
  # Rcov::RcovTask.new('coverage') do |t|
  #   t.libs << 'test'
  #   t.test_files = unit_tests + acceptance_tests
  #   t.verbose = true
  #   t.warning = true
  #   t.rcov_opts << '--sort coverage'
  #   t.rcov_opts << '--xref'
  # end

  desc "Run performance tests"
  task 'performance' do
    require File.join(File.dirname(__FILE__), 'test', 'acceptance', 'stubba_example_test')
    require File.join(File.dirname(__FILE__), 'test', 'acceptance', 'mocha_example_test')
    iterations = 1000
    puts "\nBenchmarking with #{iterations} iterations..."
    [MochaExampleTest, StubbaExampleTest].each do |test_case|
      puts "#{test_case}: #{benchmark_test_case(test_case, iterations)} seconds."
    end
  end

end

def benchmark_test_case(klass, iterations)
  require 'benchmark'
  require 'mocha/detection/mini_test'

  if defined?(MiniTest)
    minitest_version = Gem::Version.new(Mocha::Detection::MiniTest.version)
    if Gem::Requirement.new('>= 5.0.0').satisfied_by?(minitest_version)
      result = Benchmark.realtime { iterations.times { |i| klass.run(MiniTest::CompositeReporter.new) } }
      MiniTest::Runnable.runnables.delete(klass)
      result
    else
      MiniTest::Unit.output = StringIO.new
      Benchmark.realtime { iterations.times { |i| MiniTest::Unit.new.run([klass]) } }
    end
  else
    load 'test/unit/ui/console/testrunner.rb' unless defined?(Test::Unit::UI::Console::TestRunner)
    unless $silent_option
      begin
        load 'test/unit/ui/console/outputlevel.rb' unless defined?(Test::Unit::UI::Console::OutputLevel::SILENT)
        $silent_option = { :output_level => Test::Unit::UI::Console::OutputLevel::SILENT }
      rescue LoadError
        $silent_option = Test::Unit::UI::SILENT
      end
    end
    Benchmark.realtime { iterations.times { Test::Unit::UI::Console::TestRunner.run(klass, $silent_option) } }
  end
end

if ENV["MOCHA_GENERATE_DOCS"]
  require 'yard'

  desc 'Remove generated documentation'
  task 'clobber_yardoc' do
    `rm -rf ./doc`
  end

  task 'docs_environment' do
    unless ENV['GOOGLE_ANALYTICS_WEB_PROPERTY_ID']
      puts "\nWarning: GOOGLE_ANALYTICS_WEB_PROPERTY_ID was not defined\n\n"
    end
  end

  desc 'Generate documentation'
  YARD::Rake::YardocTask.new('yardoc' => 'docs_environment') do |task|
    task.options = ["--title", "Mocha #{Mocha::VERSION}"]
  end

  desc "Generate documentation"
  task 'generate_docs' => ['clobber_yardoc', 'yardoc']

  desc "Publish docs to #{MOCHA_DOCS_HOST}/docs/mocha"
  task 'publish_docs' => 'generate_docs' do
    path = "/home/freerange/docs/mocha"
    system %{ssh #{MOCHA_DOCS_HOST} "sudo rm -fr #{path} && mkdir -p #{path}" && scp -r doc/* #{MOCHA_DOCS_HOST}:#{path}}
  end
end

task 'release' => 'default' do
  Rake::Task['publish_docs'].invoke
end
