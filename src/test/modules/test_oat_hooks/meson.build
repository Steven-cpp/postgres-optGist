# Copyright (c) 2022, PostgreSQL Global Development Group

# FIXME: prevent install during main install, but not during test :/

test_oat_hooks_sources = files(
  'test_oat_hooks.c',
)

if host_system == 'windows'
  test_oat_hooks_sources += rc_lib_gen.process(win32ver_rc, extra_args: [
    '--NAME', 'test_oat_hooks',
    '--FILEDESC', 'test_oat_hooks - example use of object access hooks',])
endif

test_oat_hooks = shared_module('test_oat_hooks',
  test_oat_hooks_sources,
  kwargs: pg_mod_args,
)
testprep_targets += test_oat_hooks

tests += {
  'name': 'test_oat_hooks',
  'sd': meson.current_source_dir(),
  'bd': meson.current_build_dir(),
  'regress': {
    'sql': [
      'test_oat_hooks',
    ],
    'regress_args': ['--no-locale', '--encoding=UTF8'],
    'runningcheck': false,
  },
}
