fs = require 'fs'
path = require 'path'
spawn = require('child_process').spawn

cwd = __dirname + '/'

options =
	includeLoader: true

rjs = (cont) ->
	wrapStart = """(function(window) {
var anchosen = {};
if(typeof window.define === 'function' && window.amd){
	anchosen.define = window.define;
} else {
	anchosen.define = function(name, deps, factory){
		var params = [];
		for(var i in deps){
			var dep = deps[i];
			switch(dep){
				case 'jquery': dep = window.jQuery; break;
				case 'underscore': dep = window._; break;
				case 'knockout': dep = window.ko; break;
				case 'anchosen/view_model': dep = window.Anchosen.ViewModel; break;
				case 'anchosen/browser': dep = window.Anchosen.Browser; break;
				case 'anchosen':
				case 'anchosen/anchosen': dep = window.Anchosen; break;
				default: dep = null;
			}
			params.push(dep)
		}
		factory.apply(window, params);
	};
}"""
	wrapEnd = """
}(window));"""
	wrap = unless options.includeLoader then true else
		start: wrapStart
		end: wrapEnd
	requirejs = require 'requirejs'
	config =
		appDir: 'bin'
		dir: 'build'
		baseUrl: 'js'
		keepBuildDir: true
		skipDirOptimize: true
		optimize: "none"
		useStrict: true
		removeCombined: true
		modules: [
			{
				name: 'anchosen'
				exclude: ['knockout', 'jquery', 'underscore']
			}
		]
		namespace: 'anchosen' if options.includeLoader
		wrap: wrap
		paths:
			jquery: "jquery"
			knockout: "knockout"
			underscore: "underscore"
		shim:
			underscore:
				exports: "_"


	requirejs.optimize config, (buildResponse) ->
		config =
			appDir: 'bin'
			dir: 'build_min'
			baseUrl: 'js'
			keepBuildDir: true
			skipDirOptimize: false
			optimize: "uglify2"
			useStrict: true
			removeCombined: true
			optimizeCss: "standard"
			modules: [
				{
					name: 'anchosen'
					exclude: ['knockout', 'jquery', 'underscore']
				}
			]
			wrap: wrap
			namespace: 'anchosen' if options.includeLoader
			paths:
				jquery: "jquery"
				knockout: "knockout"
				underscore: "underscore"
			shim:
				underscore:
					exports: "_"
		requirejs.optimize config, (buildResponse) ->
			cont?()

exec = (command, args, env, cont) ->
	env.stdio = 'inherit'
	proc = spawn(command, args, env)

	proc.on 'exit', (code) ->
		console.log command if code != 0
		cont?() if code == 0

deps = (cont) ->
	console.log 'Compiling dependencies...'
	knockoutSource = fs.createReadStream cwd + 'vendor/knockout/build/output/knockout-latest.debug.js'
	knockoutDest = fs.createWriteStream cwd + 'bin/js/knockout.js'
	knockoutSource.pipe knockoutDest
	knockoutDest.on 'close', () ->
		underscoreSource = fs.createReadStream cwd + 'vendor/underscore/underscore.js'
		underscoreDest = fs.createWriteStream cwd + 'bin/js/underscore.js'
		underscoreSource.pipe underscoreDest
		underscoreDest.on 'close', () ->
			exec 'node', ['node_modules/grunt/bin/grunt'], {
				cwd: cwd + 'vendor/jquery'
			}, () ->
				fs.renameSync cwd + 'vendor/jquery/dist/jquery.js', 'bin/js/jquery.js'

				requirejsSource = fs.createReadStream cwd + '/vendor/requirejs/require.js'
				requirejsDest = fs.createWriteStream cwd + '/bin/js/require.js'
				requirejsSource.pipe requirejsDest
				requirejsDest.on 'close', () ->
					cont?()

deleteDir = (dir, cont) ->
	if fs.existsSync dir
		exec 'rm', ['-R', dir], {}, (stdout) ->
			cont?()
	else
		cont?()

clean = (cont) ->
	console.log 'Cleaning bin/ dir'
	deleteDir 'bin', () ->
		fs.mkdirSync 'bin'
		fs.mkdirSync 'bin/js'
		fs.mkdirSync 'bin/css'

		deleteDir 'build', () -> deleteDir 'build_min', () -> cont?()


build = (cont) ->
	console.log 'Building Anchosen...'
	compile_coffee () -> compile_less () ->
		cont?()

compile_less = (cont) ->
	console.log 'Compiling .less files...'
	exec 'node', ['node_modules/less/bin/lessc', 'src/less/anchosen.less', 'bin/css/anchosen.css'], {}, (stdout) ->
		cont?()

compile_coffee = (cont) ->
	console.log 'Compiling .coffee files...'
	exec 'node', ['node_modules/coffee-script/bin/coffee', '--bare', '-co', 'bin/js', 'src/coffee'], {}, (stdout) ->
		cont?()

setup = (cont) ->
	console.log 'Initting git submodules...'
	exec 'git', ['submodule', 'update', '--init', '--recursive'], {}, () -> cont?()

npm = (cont) ->
	console.log 'Running npm install...'
	if process.platform is 'win32'
		console.log 'Cannot run npm install from Cakefile under windows'
		console.log 'Please manually run "npm install" in Anchosen working directory - and vendor/jquery directory'
		console.log 'If you have done this previously, ignore this message'
		return cont?()
	exec 'npm', ['install'], {}, (stdout) ->
		exec 'npm', ['install'], { cwd: 'vendor/jquery' }, (stdout) ->
			cont?()

option '-p', '--port [PORT]', 'Sets the port number to use in the example server, defaults to 8080'


task 'all', 'compiles all of them!', (opts) ->
	if opts['no-loader']?
		options.includeLoader = false
	clean () ->
		setup () ->
			npm () ->
				deps () -> build () -> rjs()

task 'deps', 'compiles vendor libraries', () ->
	deps()

task 'build', 'compiles the project itself', () ->
	build()

task 'clean', 'cleans the bin directory', () ->
	clean()

option '-n', '--no-loader', 'Flags the build for usage in an rjs environment or not'

task 'rjs', 'RequireJSs the library', (opts) ->
	if opts['no-loader']?
		options.includeLoader = false
	build () -> rjs()

task 'setup', 'Sets up git submodules and runs npm install', () ->
	setup () -> npm()


task 'serve', 'Fire up a webserver for use with the example files', (options) ->
	port = if options.port? then parseInt options.port else 8080
	console.log "Server started on port #{port} - see http://localhost:#{port}/examples/example.html for an example run"
	exec "node", ["node_modules/coffee-script/bin/coffee", 'examples/server.coffee', "--port #{port}"], {}