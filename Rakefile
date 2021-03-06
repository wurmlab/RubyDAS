require 'rubygems'
require 'rubygems/specification' unless defined?(Gem::Specification)
require 'rake'
require 'rake/testtask'
require 'rake/packagetask'
gem 'rdoc', '=4.2.0'
require 'rdoc/rdoc'
require 'rake/task'

$:.unshift File.join(File.dirname(__FILE__), "lib")
require "rubydas/loader/gff3_fast"
require "rubydas/loader/fasta_fast"
require "rubydas/generator"

def gemspec
    @gemspec ||= begin
        Gem::Specification.load(File.expand_path('rubydas.gemspec'))
    end
end

task :default => :import

desc 'Start a console session'
task :console do
    system 'irb -I lib -r rubydas'
end

desc 'Displays the current version'
task :version do 
    puts "Current version: #{gemspec.version}"
end

desc 'Installs the gem locally'
task :install => :package do
    sh "gem install pkg/#{gemspec.name}-#{gemspec.version}"
end

task :install_deps do
    sh "bundle install"
end

desc 'Release the gem'
task :release => :package do
      sh "gem push pkg/#{gemspec.name}-#{gemspec.version}.gem"
end

Rake::PackageTask.new(gemspec, '0.0.1') do |pkg|
    pkg.need_zip = true
    pkg.need_tar = true

end

Rake::TestTask.new do |t|
    t.libs << "test"
    t.test_files = FileList['test/test.rb']
    t.verbose = true
end

Rake::TestTask.new(:live_tests) do |t|
    t.libs << "test"
    t.test_files = FileList['test/live_test.rb'] << FileList['test/live_summary_test.rb'] << FileList['test/live_results.rb']
    t.verbose = true
end

task :build_test_db do
    require "rubydas/model/feature"
    require "rubydas/model/sequence"
    require "data_mapper"
    DataMapper.setup(:default, 'sqlite:data/test.db')
    DataMapper.auto_migrate!
end

task :load_test_gff3 do
    require "rubydas/loader/gff3"
    require "data_mapper"
    DataMapper.setup(:default, 'sqlite:data/test.db')
    DataMapper.auto_upgrade!
    loader = RubyDAS::Loader::GFF3.new
    Dir.glob("test/gff3/MAL*gff3") do |name|
        loader.store name
    end
end

task :load_test_fa do
    require "rubydas/loader/fasta"
    require "data_mapper"
    DataMapper.setup(:default, 'sqlite:data/test.db')
    DataMapper.auto_upgrade!
    loader = RubyDAS::Loader::FASTA.new
    Dir.glob("test/fasta/MAL*fasta") do |name|
        loader.store name
    end
end

task :build_test_fixture => [:build_test_db, :load_test_fa, :load_test_gff3] do
    puts "Loaded test fixture"
end

#new
gff_files = Rake::FileList.new("data/*.gff")
targets = [gff_files.ext("fasta"), gff_files.ext("db")]

desc 'Start server in sub-process'
task :run_sub, [:db_name] => targets do |t, args|
    pid = Process.fork

    if pid.nil?
        Rake::Task["run"].invoke(args[:db_name])
    else
        File.open("server.pid", "w") {|f| f << pid }
        Process.detach(pid)
    end
end

desc 'Start server'
task :run, [:db_name] => targets do |t, args|
    db_name = (args[:db_name].end_with?(".db")) ? args[:db_name] : args[:db_name] << ".db"
    Dir.chdir('lib/rubydas') do
        begin
            system 'ruby server.rb ' + db_name
        rescue Interrupt
            puts "Server Stopped"
        end
    end
end

task :import, [:interval] do |t, args|
    unless args[:interval] == nil
        p args[:interval].split("=")[1].to_i
        ENV['interval'] = args[:interval].split("=")[1]
    end
    #Workaround to make sure ENV is available before rules are invoked
    Rake::Task['make_db'].invoke
end

task :make_db => targets

task :clean, [:name] do |t, args|
    sh "rm data/#{args[:name]}.fasta"
    sh "rm data/#{args[:name]}.db"
    sh "rm -rf public/#{args[:name]}"
end

rule '.fasta' => ['.gff'] do |t|    
    fasta = File.open(t.name, "w")

    reached = false
    File.open(t.source, "r").each_line do |line|
        if !reached && line.chomp == "##FASTA"
            reached = true
            next
        end
        if reached
            fasta.write(line)
        end
    end
end

rule '.db' => ['.gff'] do |t|
    str = "Name : #{t.name}, Source: #{t.source}"

    db_path = 'sqlite://' + File.expand_path(t.name)
    gff_path = File.expand_path(t.source)
    fasta_path = gff_path.chomp(".gff").concat(".fasta")
    public_folder = t.name.chomp(".db")

    DataMapper.setup(:default, db_path)
    DataMapper.auto_migrate!

    RubyDAS::Loader::GFF3Fast.new(gff_path, ENV['interval']).store

    DataMapper.auto_upgrade!
    RubyDAS::Loader::FASTAFast.new(fasta_path, ENV['interval']).store

    Generator.new(public_folder).create_public_folder

    #Can be removed, doc says this explicitly stores some relationships which are
    #otherwise lazily evaluated. We might not even have any
    DataMapper.finalize
end

