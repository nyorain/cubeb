project('cubeb', ['c', 'cpp'],
	license: 'ISC',
	version: '0.1.0',
	meson_version: '>=0.43',
	default_options: [
  		'warning_level=2',
  		'werror=false'])

shared = get_option('shared')

# don't use default_options to allow being embedded as subproject
add_project_arguments(['-std=gnu++11'], language: 'cpp')
add_project_link_arguments(['-std=gnu++11'], language: 'cpp')
add_project_arguments(['-std=gnu11'], language: 'c')
add_project_link_arguments(['-std=gnu11'], language: 'c')

# or use build_machine?
cc = meson.get_compiler('c')
cppc = meson.get_compiler('cpp')

windows = host_machine.system() == 'windows'
android = (cc.get_define('__ANDROID__') != '')
darwin = host_machine.system() == 'darwin'

# warnings and stuff
warnings = [
	'-Wno-unused-parameter',
	'-Wno-gnu-zero-variadic-macro-arguments',
]

add_project_arguments(
	cc.get_supported_arguments(warnings),
	language: 'c')
add_project_arguments(
	cppc.get_supported_arguments(warnings),
	language: 'cpp')

export_arg = ''
cubeb_args = []
if shared
	target_type = 'shared_library'
	if windows
		export_arg = '__declspec(dllexport)'
		cubeb_args += '-DCUBEB_EXPORT=__declspec(dllexport)'
	endif
else
	target_type = 'static_library'
endif


# cubeb
cubeb_inc = include_directories(
	'.',
	'include',
	'src',
)

cubeb_src = files(
	'src/cubeb.c',
	'src/cubeb_mixer.cpp',
	'src/cubeb_resampler.cpp',
	'src/cubeb_log.cpp',
	'src/cubeb_strings.c',
	'src/cubeb_utils.cpp',
	'src/speex/resample.c',
)

cubeb_args += [
	'-DOUTSIDE_SPEEX',
	'-DFLOATING_POINT',
	'-DEXPORT=' + export_arg,
	'-DRANDOM_PREFIX=speex',
]

dep_threads = dependency('threads', required: true)
dep_dl = cc.find_library('dl', required: false)
dep_m = cc.find_library('m', required: false)

cubeb_deps = [
	dep_threads,
	dep_dl,
	dep_m,
]

conf_data = configuration_data()
conf_data.set('CUBEB_SHARED', shared)
configure_file(input: 'cubeb_export.h.in',
	output: 'cubeb_export.h',
	install_dir: 'include/cubeb',
	configuration: conf_data)

# check backends
dep_pulse = dependency('libpulse', required: false)
if dep_pulse.found()
	cubeb_deps += dep_pulse
	cubeb_src += files('src/cubeb_pulse.c')
	cubeb_args += '-DUSE_PULSE'
endif

dep_alsa = dependency('alsa', required: false)
if dep_alsa.found()
	cubeb_deps += dep_alsa
	cubeb_src += files('src/cubeb_alsa.c')
	cubeb_args += '-DUSE_ALSA'
endif

dep_jack = dependency('jack', required: false)
if dep_jack.found()
	cubeb_deps += dep_jack
	cubeb_src += files('src/cubeb_jack.cpp')
	cubeb_args += '-DUSE_JACK'
endif

if android
	lib_opensl = cc.find_library('OpenSLES', required: false)
	if cc.has_header('SLES/OpenSLES.h') and lib_opensl.found()
		cubeb_deps += [lib_opensl]
		cubeb_src += files(
			'src/cubeb_opensl.c',
			'src/cubeb-jni.cpp',
		)
		cubeb_args += '-DUSE_OPENSL'
	endif

	lib_log = cc.find_library('log')
	if lib_log.found()
		cubeb_deps += [lib_log]
		cubeb_src += files('src/cubeb_audiotrack.c')
		cubeb_args += '-DUSE_AUDIOTRACK'
	endif

	# we don't actually need the library since we dlsym but
	# we use it to check if aaudio is available
	lib_aaudio = cc.find_library('aaudio')
	if lib_aaudio.found()
		cubeb_src += files('src/cubeb_aaudio.cpp')
		cubeb_args += '-DUSE_AAUDIO'

		# set this definition to enable low latency mode. Possibly bad for battery
		cubeb_args += '-DCUBEB_AAUDIO_LOW_LATENCY'

		# set this definition to enable power saving mode. Possibly resulting
		# in high latency
		# cubeb_args += '-DCUBEB_AAUDIO_LOW_POWER_SAVING'

		# set this mode to make the backend use an exclusive stream.
		# will decrease latency.
		# cubeb_args += '-DCUBEB_AAUDIO_EXCLUSIVE_STREAM'
	endif
endif

if windows
	lib_avrt = cc.find_library('avrt', required: false)
	lib_ole32 = cc.find_library('ole32', required: false)
	if cc.has_header('audioclient.h') and lib_avrt.found() and lib_ole32.found()
		cubeb_deps += [lib_avrt, lib_ole32]
		cubeb_src += files('src/cubeb_wasapi.cpp')
		cubeb_args += '-DUSE_WASAPI'
	endif

	lib_winmm = cc.find_library('winmm', required: false)
	if cc.has_header('windows.h') and lib_winmm.found()
		cubeb_deps += [lib_winmm]
		cubeb_src += files('src/cubeb_winmm.c')
		cubeb_args += '-DUSE_WINMM'
	endif
endif

if darwin
	# TODO: no idea if this is correct
	dep_applestuff += dependency('applestuff', modules: [
		'AudioUnit',
		'CoreAudio',
		'CoreServices',
	], required: false)
	if cc.has_header('AudioUnit/AudioUnit.h') and dep_applestuff.found()
		cubeb_src += files(
			'src/cubeb_audiounit.cpp',
			'src/cubeb_osx_run_loop.cpp',
		)
		cubeb_deps += dep_applestuff
		cubeb_args += '-DUSE_AUDIOUNIT'
	endif
endif

if cc.has_header('sys/audioio.h')
	cubeb_src += files('src/cubeb_sun.c')
	cubeb_args += '-DUSE_SUN'
endif

lib_kai = cc.find_library('kai', required: false)
if cc.has_header('kai.h') and lib_kai.found()
	cubeb_deps += lib_kai
	cubeb_src += files('src/cubeb_kai.c')
	cubeb_args += '-DUSE_KAI'
endif

# build library
cubeb_lib = build_target('cubeb',
	cubeb_src,
	dependencies: cubeb_deps,
	include_directories: cubeb_inc,
	c_args: cubeb_args,
	cpp_args: cubeb_args,
	install: true,
	target_type: target_type)

# declare dependency
cubeb_dep = declare_dependency(
	link_with: cubeb_lib,
	include_directories: cubeb_inc,
)
