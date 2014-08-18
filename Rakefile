require 'openssl'
require 'io/console'
require 'colorize'

API_TOKEN = 'YOUR_TESTFLIGHT_API_TOKEN'
TEAM_TOKEN = 'YOUR_TESTFLIGHT_TEAM_TOKEN'

PROJECT_SCHEME = 'FXFormsCoreData' #
PROJECT_NAME = 'FXFormsCoreData.xcodeproj'
WORKSPACE_NAME = 'FXFormsCoreData.xcworkspace'
CONFIGURATION = 'Release' # The build configuration to use
INFOPLIST_PATH = 'FXFormsCoreData/FXFormsCoreData-Info.plist'
IPA_FILE_NAME = 'FXFormsCoreData.ipa'
RESIGNED_IPA_FILE_NAME = 'FXFormsCoreData.resigned.ipa'
DSYM_FILE_NAME = 'FXFormsCoreData.app.dSYM.zip'

DISTRIBUTION_CERT_FILE = "provisioning/distribution.p12"
DISTRIBUTION_CERT_NAME_FILE = "provisioning/distributionCertName.txt"

desc 'Checks TestFlight Tokens'
task :check_tokens do
  if (API_TOKEN == 'YOUR_TESTFLIGHT_API_TOKEN')
    abort("Error : You need to set API_TOKEN located on top section of Rakefile".colorize(:red))
  end

  if (API_TOKEN == 'YOUR_TESTFLIGHT_TEAM_TOKEN')
    abort("Error : You need to set TEAM_TOKEN located on top section of Rakefile".colorize(:red))
  end
end

desc 'Checks p12 file existence'
task :check_p12 do
  if (!File.exists?(DISTRIBUTION_CERT_FILE))
    abort("Error : You need to have #{DISTRIBUTION_CERT_FILE} corresponding to your distribution certificate".colorize(:red))
  end
end

def registered_distribution_cert_name
  return "" if !File.exists?(DISTRIBUTION_CERT_NAME_FILE)
  return File.read(DISTRIBUTION_CERT_NAME_FILE).chomp
end

def distribution_cert_registered?
  $cert_name = registered_distribution_cert_name
  results = `security find-identity -v -p codesigning | awk '{$1=$2=""; print $0}'`
  results.split(/\n/).each do |result|
    cert_name = result.gsub(/  \"/, '').gsub(/\"/, '')
    if (cert_name == $cert_name)
      return true
    end
  end
  false
end

def install_distribution_cert(password)
  system "security unlock -p login ~/Library/Keychains/login.keychain"
  system "security import #{DISTRIBUTION_CERT_FILE} -k ~/Library/Keychains/login.keychain -P #{password} -T /usr/bin/codesign"
end

def register_distribution_cert_name(password)
  raw = File.read(DISTRIBUTION_CERT_FILE)
  certificate = OpenSSL::PKCS12.new(raw, password).certificate
  names = certificate.subject.to_a
  cert_name_a = names.select {|n| n.first == "CN" }.first
  $cert_name = cert_name_a[1]
  puts $cert_name

  FileUtils.remove DISTRIBUTION_CERT_NAME_FILE, :force => true
  system "echo '#{$cert_name}' > #{DISTRIBUTION_CERT_NAME_FILE}"
  FileUtils.chmod 0444, DISTRIBUTION_CERT_NAME_FILE
end

desc "Install p12 file"
task :install_p12 do

  if (!distribution_cert_registered?)
    puts "Installing distribution certificate...".green

    STDOUT.puts "Please enter password for Distribution certificate p12 file > ".green.underline
    password = STDIN.noecho(&:gets).strip

    install_distribution_cert(password)

    register_distribution_cert_name(password)

    puts "Successfully installed distribution certificate!".green
  end
end

desc 'Download Adhoc Profile'
task :adhoc_profile do
  sh "rm -f *.mobileprovision"

  if (!File.exists?("provisioning/adhoc.mobileprovision"))
    puts "Downloading provisioning profile for adhoc distribution".green

    STDOUT.puts "Please enter username (Apple ID) for your iOS developer account > ".green.underline
    username = STDIN.gets.strip
    sh "ios profiles:download --type distribution --username #{username}"
    provisions = Dir.entries(".").select {|f| File.extname(f) == ".mobileprovision"}
    FileUtils.move provisions.first, 'provisioning/adhoc.mobileprovision', :force => true

    puts "Successfully downloaded adhoc distribution profile!".green
  end
end

desc 'Build IPA'
task :build_ipa => ['check_tokens','check_p12','install_p12','adhoc_profile'] do
  puts "Attempting to build IPA...".green
  sh "rm -rf build/*.ipa"
  sh "rm -rf build/*.app.dSYM.zip"
  sh "ipa build --destination 'build' --scheme #{PROJECT_SCHEME} --workspace #{WORKSPACE_NAME} --project #{PROJECT_NAME} --configuration #{CONFIGURATION} --clean /tmp"
  result = `scripts/ipa_sign build/#{IPA_FILE_NAME} provisioning/adhoc.mobileprovision '#{$cert_name}'`
  FileUtils.move RESIGNED_IPA_FILE_NAME, "build/#{IPA_FILE_NAME}", :force => true
  puts "Successfully build IPA at build/#{IPA_FILE_NAME}!".green
end

desc "Rev the build numbers in a project's plist"
task :rev do
  puts "Attempting to update #{INFOPLIST_PATH} build version...".green
  oldBuildNumber = `/usr/libexec/PlistBuddy -c "Print CFBundleVersion" #{INFOPLIST_PATH}`.chomp

  newBuildNumber = Integer(oldBuildNumber) + 1
  `/usr/libexec/PlistBuddy -c "Set :CFBundleVersion #{newBuildNumber}" #{INFOPLIST_PATH}`.chomp

  versionString = `/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" #{INFOPLIST_PATH}`.chomp

  puts "Successfully updated from #{versionString} (#{oldBuildNumber}) to #{versionString} (#{newBuildNumber})".green
end

desc "Commit git changes"
task :push do
  build_number = `/usr/libexec/PlistBuddy -c "Print CFBundleVersion" #{INFOPLIST_PATH} | tr -d "\n"`
  version = `/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" #{INFOPLIST_PATH} | tr -d "\n"`
  sh "git add #{INFOPLIST_PATH}"
  sh "git commit -m 'Release #{version} (#{build_number})'"
  sh "git tag -a b#{version}-#{build_number} -m 'Release #{version} (#{build_number})'"
  sh "git push origin master --tags"

  puts "Successfully tagged current release version and pushed to remote repo".green
end

desc 'Deploy Release Build to TestFlight'
task :release => ['rev','build_ipa'] do
  puts "Attempting to release build to TestFlight...".green
  sh "ipa distribute:testflight --api_token #{API_TOKEN} --team_token #{TEAM_TOKEN} --file build/#{IPA_FILE_NAME} --dsym build/#{DSYM_FILE_NAME}"
  Rake::Task['push'].execute
end
