#-*- mode: ruby -*-

gemspec :jar => 'jopenssl', :include_jars => true

sonatype_url = 'https://oss.sonatype.org/content/repositories/snapshots/'
snapshot_repository :id => 'sonatype', :url => sonatype_url

if model.version.to_s.match /[a-zA-Z]/

  # deploy snapshot versions on sonatype !!!
  model.version = model.version + '-SNAPSHOT'
  plugin :deploy, '2.8.1' do
    execute_goals( :deploy,
                   :skip => false,
                   :altDeploymentRepository => "sonatype-nexus-snapshots::default::#{sonatype_url}" )
  end
end

java_target = '1.6'
gen_sources = '${basedir}/target/generated-sources' # hard-coded in AnnotationBinder

plugin( 'org.codehaus.mojo:exec-maven-plugin', '1.3.2' ) do

=begin
  invoker_main  = '-Djruby.bytecode.version=${compiler.target}'
  #invoker_main << ' -classpath '
  invoker_main << ' org.jruby.anno.InvokerGenerator'
  invoker_main << " #{gen_sources}/annotated_classes.txt ${project.build.outputDirectory}"

  dependency 'org.jruby', 'jruby-core', '1.7.13'

  execute_goal :java, :id => 'invoker-generator', :phase => 'process-classes',
      :mainClass => 'org.jruby.anno.InvokerGenerator', :classpathScope => 'compile',
      #:arguments => [ '${gen.sources}/annotated_classes.txt', '${project.build.outputDirectory}' ] do
      :commandlineArgs => "#{gen_sources}/annotated_classes.txt ${project.build.outputDirectory}",
      :classpathScope => 'runtime', :additionalClasspathElements => [ '${project.build.outputDirectory}' ],
      :includeProjectDependencies => false, :includePluginDependencies => true do

    #systemProperties do
    #  property '-Djruby.bytecode.version=${compiler.target}'
    #end
=end

  execute_goal :exec, :id => 'invoker-generator', :phase => 'process-classes',
      :executable => 'java', :classpathScope => 'compile',
      :arguments => [ "-Djruby.bytecode.version=#{java_target}",
                      '-classpath', xml( '<classpath/>' ),
                      'org.jruby.anno.InvokerGenerator',
                      "#{gen_sources}/annotated_classes.txt",
                      '${project.build.outputDirectory}' ]
end

plugin( 'org.codehaus.mojo:build-helper-maven-plugin', '1.9' ) do
  execute_goal 'add-source', :phase => 'process-classes', :sources => [ gen_sources ]
end

plugin( :compiler, '3.1',
        :source => java_target, :target => java_target,
        :encoding => 'UTF-8', :debug => true,
        :showWarnings => true, :showDeprecation => true,

        :generatedSourcesDirectory => gen_sources,
        :annotationProcessors => [ 'org.jruby.anno.AnnotationBinder' ],
        :compilerArgs => [ '-XDignore.symbol.file=true' ] ) do

  #execute_goal :compile, :id => 'annotation-binder', :phase => 'compile',
  #    :generatedSourcesDirectory => gen_sources, #:outputDirectory => gen_sources,
  #    :annotationProcessors => [ 'org.jruby.anno.AnnotationBinder' ],
  #    :proc => 'only', # :compilerReuseStrategy => 'alwaysNew',
  #    :useIncrementalCompilation => false, :fork => true, :verbose => true,
  #    :compilerArgs => [ '-XDignore.symbol.file=true', '-J-Dfile.encoding=UTF-8' ]

  execute_goal :compile, :id => 'compile-populators', :phase => 'process-classes',
      :includes => [ 'org/jruby/gen/**/*.java' ], :optimize => true,
      :compilerArgs => [ '-XDignore.symbol.file=true' ]
      # NOTE: maybe '-J-Xbootclasspath/p:${unsafe.jar}' ... as well ?!
end

# NOTE: unfortunately we can not use 1.6.8 to generate invokers ...
# although we'd like to compile against 1.6 to make sure all is well
jar 'org.jruby:jruby-core', '1.7.13', :scope => :provided   # 1.6.8
jar 'junit:junit', '4.11', :scope => :test

jruby_plugin! :gem do
  # when installing dependent gems we want to use the built in openssl
  # not the one from this lib directory
  execute_goal :id => 'default-initialize', :libDirectory => 'something-which-does-not-exists'
  execute_goals :id => 'default-push', :skip => true
end

supported_bc_versions = [ '1.47', '1.48', '1.49', '1.50' ]

properties( 'jruby.plugins.version' => '1.0.5',
            'jruby.versions' => '1.7.13',
            'bc.versions' => supported_bc_versions.last,
            'invoker.test' => '${bc.versions}',
            # allow to skip all tests with -Dmaven.test.skip
            'invoker.skip' => '${maven.test.skip}',
            'runit.dir' => 'src/test/ruby/**/test_*.rb',
            # use this version of jruby for ALL the jruby-maven-plugins
            'jruby.version' => '1.7.13',
            # dump pom.xml as readonly when running 'rmvn'
            'tesla.dump.pom' => 'pom.xml',
            'tesla.dump.readonly' => true )

phase :package do # we need to assure the jars are installed in ./lib
  jruby_plugin(:runit) { execute_goal( :test, :runitDirectory => '${runit.dir}' ) }
end

invoker_run_options = {
    :id => 'tests-with-different-bc-versions',
    :projectsDirectory => 'integration',
    :pomIncludes => [ '*/pom.xml' ],
    :streamLogs => true,
    # pass those properties on to the test project
    :properties => {
      'jruby.versions' => '${jruby.versions}',
      'jruby.modes' => '${jruby.modes}',
      'jruby.openssl.version' => '${project.version}',
      'bc.versions' => '${bc.versions}',
      'runit.dir' => '${runit.dir}' }
}

profile :id => 'test-1.6.8' do
  plugin :invoker, '1.8' do
    execute_goals( :install, :run, invoker_run_options )
  end
  properties 'jruby.versions' => '1.6.8', 'jruby.modes' => '1.8,1.9',
             'bc.versions' => supported_bc_versions.join(',')
end

profile :id => 'test-1.7.4' do
  plugin :invoker, '1.8' do
    execute_goals( :install, :run, invoker_run_options )
  end
  properties 'jruby.versions' => '1.7.4', 'jruby.modes' => '1.8,1.9,2.0',
             'bc.versions' => supported_bc_versions.join(',')
end

profile :id => 'test-1.7.13' do
  plugin :invoker, '1.8' do
    execute_goals( :install, :run, invoker_run_options )
  end
  properties 'jruby.versions' => '1.7.13', 'jruby.modes' => '1.8,1.9,2.0',
             'bc.versions' => supported_bc_versions.join(',')
end

profile :id => 'test-1.7.15' do
  plugin :invoker, '1.8' do
    execute_goals( :install, :run, invoker_run_options )
  end
  properties 'jruby.versions' => '1.7.15', 'jruby.modes' => '1.8,1.9,2.0',
             'bc.versions' => supported_bc_versions.join(',')
end

profile :id => 'test-9000' do
  plugin :invoker, '1.8' do
    execute_goals( :install, :run, invoker_run_options )
  end
  properties 'jruby.version' => '9000.dev-SNAPSHOT',
             'jruby.versions' => '9000.dev-SNAPSHOT',
             'bc.versions' => supported_bc_versions.join(',')
end

# vim: syntax=Ruby
