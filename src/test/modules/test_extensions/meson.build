# Copyright (c) 2022, PostgreSQL Global Development Group

# FIXME: prevent install during main install, but not during test :/
install_data(
  'test_ext1--1.0.sql',
  'test_ext1.control',
  'test_ext2--1.0.sql',
  'test_ext2.control',
  'test_ext3--1.0.sql',
  'test_ext3.control',
  'test_ext4--1.0.sql',
  'test_ext4.control',
  'test_ext5--1.0.sql',
  'test_ext5.control',
  'test_ext6--1.0.sql',
  'test_ext6.control',
  'test_ext7--1.0--2.0.sql',
  'test_ext7--1.0.sql',
  'test_ext7.control',
  'test_ext8--1.0.sql',
  'test_ext8.control',
  'test_ext_cine--1.0.sql',
  'test_ext_cine--1.0--1.1.sql',
  'test_ext_cine.control',
  'test_ext_cor--1.0.sql',
  'test_ext_cor.control',
  'test_ext_cyclic1--1.0.sql',
  'test_ext_cyclic1.control',
  'test_ext_cyclic2--1.0.sql',
  'test_ext_cyclic2.control',
  'test_ext_evttrig--1.0--2.0.sql',
  'test_ext_evttrig--1.0.sql',
  'test_ext_evttrig.control',
  kwargs: contrib_data_args,
)

tests += {
  'name': 'test_extensions',
  'sd': meson.current_source_dir(),
  'bd': meson.current_build_dir(),
  'regress': {
    'sql': [
      'test_extensions',
      'test_extdepend',
    ],
  },
}
