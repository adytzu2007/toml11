project('toml11', ['cpp'],
  version         : '3.0.1',
  default_options : ['cpp_std=c++11']
)

toml11_inc = include_directories('.')

toml11_dep = declare_dependency(
  include_directories : toml11_inc,
  version             : meson.project_version(),
)
