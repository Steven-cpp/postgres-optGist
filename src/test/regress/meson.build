# Copyright (c) 2022, PostgreSQL Global Development Group

# also used by isolationtester and ecpg tests
pg_regress_c = files('pg_regress.c')
pg_regress_inc = include_directories('.')

regress_sources = pg_regress_c + files(
  'pg_regress_main.c'
)

# Need make up something roughly like x86_64-pc-mingw64. resultmap matches on
# patterns like ".*-.*-mingw.*". We probably can do better, but for now just
# replace 'gcc' with 'mingw' on windows.
host_tuple_cc = cc.get_id()
if host_system == 'windows' and host_tuple_cc == 'gcc'
  host_tuple_cc = 'mingw'
elif host_system == 'cygwin' and host_tuple_cc == 'gcc'
  host_tuple_cc = 'cygwin'
endif
host_tuple = '@0@-@1@-@2@'.format(host_cpu, host_system, host_tuple_cc)

pg_regress_cflags = ['-DHOST_TUPLE="@0@"'.format(host_tuple), '-DSHELLPROG="/bin/sh"']

if host_system == 'windows'
  regress_sources += rc_bin_gen.process(win32ver_rc, extra_args: [
    '--NAME', 'pg_regress',
    '--FILEDESC', 'pg_regress - test driver',])
endif

pg_regress = executable('pg_regress',
  regress_sources,
  c_args: pg_regress_cflags,
  dependencies: [frontend_code],
  kwargs: default_bin_args + {
    'install_dir': dir_pgxs / 'src/test/regress',
  },
)
bin_targets += pg_regress

regress_module = shared_module('regress',
  ['regress.c'],
  kwargs: pg_mod_args + {
    'install': false,
  },
)
testprep_targets += regress_module

# Get some extra C modules from contrib/spi but mark them as not to be
# installed.
# FIXME: avoid the duplication.

autoinc_regress = shared_module('autoinc',
  ['../../../contrib/spi/autoinc.c'],
  kwargs: pg_mod_args + {
    'install': false,
  },
)
testprep_targets += autoinc_regress

refint_regress = shared_module('refint',
  ['../../../contrib/spi/refint.c'],
  c_args: refint_cflags,
  kwargs: pg_mod_args + {
    'install': false,
  },
)
testprep_targets += refint_regress


tests += {
  'name': 'regress',
  'sd': meson.current_source_dir(),
  'bd': meson.current_build_dir(),
  'regress': {
    'schedule': files('parallel_schedule'),
    'test_kwargs': {
      'priority': 50,
      'timeout': 1000,
    },
    'dbname': 'regression',
  },
}
