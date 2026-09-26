# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'

target 'Nally' do
  # Comment the next line if you don't want to use dynamic frameworks
  use_frameworks!
  platform :osx, '12.0'
  # Pods for Nally
  pod 'ImgurAnonymousAPIClient', :git => 'https://github.com/nolanw/ImgurAnonymousAPIClient.git', :tag => 'v0.3.2'

end

target 'TextSuiteTests' do
  # Comment the next line if you don't want to use dynamic frameworks
  use_frameworks!
  platform :osx, '12.0'

  # Pods for TextSuiteTests

end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      # Xcode 27 only supports deployment targets >= 12.0; the pods ship older ones.
      config.build_settings['MACOSX_DEPLOYMENT_TARGET'] = '12.0'
    end
  end

  # AFNetworking 4.0.1 imports <netinet6/in6.h> directly. The current SDK marks
  # that header private to the Darwin module, which is a hard error under
  # -fmodules. netinet/in.h (imported on the line above) already provides it,
  # so the import is simply dropped.
  Dir.glob(File.join(installer.sandbox.root, 'AFNetworking', '**', '*.m')).each do |file|
    source = File.read(file)
    patched = source.gsub(/^#import <netinet6\/in6\.h>\n/, '')
    next if patched == source
    mode = File.stat(file).mode
    File.chmod(0644, file)
    File.write(file, patched)
    File.chmod(mode, file)
  end
end
