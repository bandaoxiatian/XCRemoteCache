# encoding: utf-8
require_relative 'tasks/e2e'

################################
# Rake configuration
################################

# Paths
DERIVED_DATA_DIR = File.join('.build').freeze
RELEASES_ROOT_DIR = File.join('releases').freeze

EXECUTABLE_NAME = 'XCRemoteCache'
EXECUTABLE_NAMES = ['xclibtool', 'xcpostbuild', 'xcprebuild', 'xcprepare', 'xcswiftc', 'xcld', 'xcldplusplus'] 
PROJECT_NAME = 'XCRemoteCache'

SWIFTLINT_ENABLED = true
SWIFTFORMAT_ENABLED = true

################################
# Tasks
################################

task :prepare do
  Dir.mkdir(DERIVED_DATA_DIR) unless File.exists?(DERIVED_DATA_DIR)
end

desc 'lint'
task :lint => [:prepare] do
  puts 'Run linting'

  system("swiftformat --lint --config .swiftformat --cache ignore .") or abort "swiftformat failure" if SWIFTFORMAT_ENABLED
  system("swiftlint lint --config .swiftlint.yml --strict") or abort "swiftlint failure" if SWIFTLINT_ENABLED
end

task :autocorrect => [:prepare]  do 
  puts 'Run autocorrect'

  system("swiftformat --config .swiftformat --cache ignore .") or abort "swiftformat failure" if SWIFTFORMAT_ENABLED
  system("swiftlint autocorrect --config .swiftlint.yml") or abort "swiftlint failure" if SWIFTLINT_ENABLED
end

desc 'build package artifacts'
task :build, [:configuration, :arch, :sdks, :is_archive] do |task, args|
  # Set task defaults
  args.with_defaults(:configuration => 'debug', :sdks => ['macos'])

  unless args.configuration == 'Debug'.downcase || args.configuration == 'Release'.downcase
    fail("Unsupported configuration. Valid values: ['Debug', 'Release']. Found '#{args.configuration}''")
  end

  # Clean data generated by SPM
  # FIXME: dangerous recursive rm
  system("rm -rf #{DERIVED_DATA_DIR} > /dev/null 2>&1")

  # Build
  build_paths = []
  args.sdks.each do |sdk|
    spm_build(args.configuration, args.arch)

    # Path of the executable looks like: `.build/(debug|release)/XCRemoteCache`
    build_path_base = File.join(DERIVED_DATA_DIR, args.configuration)
    sdk_build_paths = EXECUTABLE_NAMES.map {|e| File.join(build_path_base, e)}

    build_paths.push(sdk_build_paths)
  end

  puts "Build products: #{build_paths}"

  if args.configuration == 'Release'.downcase
    puts "Creating release zip"
    create_release_zip(build_paths[0])
  end
end

desc 'run tests with SPM'
task :test do
  # Running tests
  spm_test()
end

desc 'build and run E2E tests'
task :e2e => [:build, :e2e_only]

desc 'run E2E tests without building the XCRemoteCache binary'
task :e2e_only => ['e2e:run']

################################
# Helper functions
################################

def spm_build(configuration, arch)
  spm_cmd = "swift build "\
              "-c #{configuration} "\
              "#{arch.nil? ? "" : "--triple #{arch}"} "
  system(spm_cmd) or abort "Build failure"
end

def bash(command)
  system "bash -c \"#{command}\""
end

def spm_test()
  tests_output_file = File.join(DERIVED_DATA_DIR, 'tests.log')
  # Redirect error stream with to a file and pass to the second stream output 
  spm_cmd = "swift test --enable-code-coverage 2> >(tee #{tests_output_file})"
  test_succeeded = bash(spm_cmd)
  
  abort "Test failure" unless test_succeeded
end

def create_release_zip(build_paths)
  release_dir = RELEASES_ROOT_DIR
  
  # Create and move files into the release directory
  mkdir_p release_dir
  build_paths.each {|p|
    cp_r p, release_dir
  }
  
  output_artifact_basename = "#{PROJECT_NAME}.zip"

  Dir.chdir(release_dir) do
    # -X: no extras (uid, gid, file times, ...)
    # -x: exclude .DS_Store
    # -r: recursive
    system("zip -X -x '*.DS_Store' -r #{output_artifact_basename} .") or abort "zip failure"
    # List contents of zip file
    system("unzip -l #{output_artifact_basename}") or abort "unzip failure"
  end
end
