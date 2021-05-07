import os
import sys
import glob
import shutil
import subprocess
import excons
import excons.tools.zlib as zlib
import excons.tools.threads as threads
import excons.tools.python as python
import excons.tools.boost as boost
import SCons.Script # pylint: disable=import-error


excons.SetArgument("use-c++11", 1)


env = excons.MakeBaseEnv()


lib_version = (2, 2, 2)
lib_version_str = "%d.%d.%d" % lib_version
lib_suffix = excons.GetArgument("openexr-suffix", "-2_2")
#static_lib_suffix = lib_suffix + excons.GetArgument("openexr-static-suffix", "_s")
namespace_version = (excons.GetArgument("openexr-namespace-version", 1, int) != 0)
zlib_win_api = (excons.GetArgument("openexr-zlib-winapi", 0, int) != 0)
pyilmbase_static = (excons.GetArgument("ilmbase-python-staticlibs", 1, int) != 0)
have_gcc_include_asm_avx = False
have_sysconf_nprocessors_onln = False

gcc_include_asm_avx_check_src = """
int main()
{
#if defined(__GNUC__) && defined(__SSE2__) 
   int n   = 0;
   int eax = 0;
   int edx = 0;
   __asm__(
      \"xgetbv     ;\"
      \"vzeroupper  \"
      : \"=a\"(eax), \"=d\"(edx) : \"c\"(n) : );
#else
   #error No GCC style inline asm supported for AVX instructions
#endif
}
"""

_sc_nprocessors_onln_check_src = """
#include <unistd.h>
int main()
{
   sysconf(_SC_NPROCESSORS_ONLN);
}
"""

def CheckConfigStatus(path):
   if not os.path.isfile(path):
      return True
   else:
      with open(path, "r") as f:
         for line in f.readlines():
            spl = line.strip().split(" ")
            if spl[0] == "namespace_version":
               val = (int(spl[1]) != 0)
               if val != namespace_version:
                  return True
            elif spl[0] == "platform":
               if spl[1] != sys.platform:
                  return True
            elif spl[0] == "have_gcc_include_asm_avx":
               val = (int(spl[1]) != 0)
               if val != have_gcc_include_asm_avx:
                  return True
            elif spl[0] == "have_sysconf_nprocessors_onln":
               val = (int(spl[1]) != 0)
               if val != have_sysconf_nprocessors_onln:
                  return True
      return False

def WriteConfigStatus(path):
   with open(path, "w") as f:
      f.write("namespace_version %d\n" % namespace_version)
      f.write("platform %s\n" % sys.platform)
      f.write("have_gcc_include_asm_avx %d\n" % have_gcc_include_asm_avx)
      f.write("have_sysconf_nprocessors_onln %d\n" % have_sysconf_nprocessors_onln)

def GenerateIlmBaseConfig(config_header):   
   update = False

   if not os.path.isfile(config_header):
      update = True
   else:
      update = CheckConfigStatus("ilmbase_config.status")
      if update:
         os.remove(config_header)

   if update:
      print("Update '%s'..." % os.path.basename(config_header))

      WriteConfigStatus("ilmbase_config.status")

      d = os.path.dirname(config_header)
      if not os.path.isdir(d):
         os.makedirs(d)

      if sys.platform == "win32":
         shutil.copy("IlmBase/config.windows/IlmBaseConfig.h", config_header)
      else:
         with open(config_header, "w") as f:
            f.write("#define HAVE_PTHREAD 1\n")
            if sys.platform != "darwin":
               f.write("#define ILMBASE_HAVE_LARGE_STACK 1\n")
               f.write("#define HAVE_POSIX_SEMAPHORES 1\n")
               f.write("#define ILMBASE_HAVE_CONTROL_REGISTER_SUPPORT 1\n")
            f.write("\n")
      
      with open(config_header, "a") as f:
         if namespace_version:
            api_version = "%s_%s" % (lib_version[0], lib_version[1])
            f.write("#define ILMBASE_INTERNAL_NAMESPACE_CUSTOM 1\n")
            f.write("#define IMATH_INTERNAL_NAMESPACE Imath_%s\n" % api_version)
            f.write("#define IEX_INTERNAL_NAMESPACE Iex%s\n" % api_version)
            f.write("#define ILMTHREAD_INTERNAL_NAMESPACE IlmThread_%s\n" % api_version)
         else:
            f.write("#define ILMBASE_INTERNAL_NAMESPACE_CUSTOM 0\n")
            f.write("#define IMATH_INTERNAL_NAMESPACE Imath\n")
            f.write("#define IEX_INTERNAL_NAMESPACE Iex\n")
            f.write("#define ILMTHREAD_INTERNAL_NAMESPACE IlmThread\n")
         f.write("\n")

         f.write("#define IMATH_NAMESPACE Imath\n")
         f.write("#define IEX_NAMESPACE Iex\n")
         f.write("#define ILMTHREAD_NAMESPACE IlmThread\n")
         f.write("\n")

         f.write("#define ILMBASE_VERSION_STRING \"%d.%d.%d\"\n" % lib_version)
         f.write("#define ILMBASE_PACKAGE_STRING \"IlmBase %d.%d.%d\"\n" % lib_version)
         f.write("#define ILMBASE_VERSION_MAJOR %d\n" % lib_version[0])
         f.write("#define ILMBASE_VERSION_MINOR %d\n" % lib_version[1])
         f.write("#define ILMBASE_VERSION_PATCH %d\n" % lib_version[2])
         f.write("\n")

         f.write("// Version as a single hex number, e.g. 0x01000300 == 1.0.3\n")
         f.write("#define ILMBASE_VERSION_HEX ((ILMBASE_VERSION_MAJOR << 24) | \\\n")
         f.write("                             (ILMBASE_VERSION_MINOR << 16) | \\\n")
         f.write("                             (ILMBASE_VERSION_PATCH <<  8))\n")
         f.write("\n")

def GenerateOpenEXRConfig(config_header):
   update = False

   if not os.path.isfile(config_header):
      update = True
   else:
      update = CheckConfigStatus("openexr_config.status")
      if update:
         os.remove(config_header)

   if update:
      print("Update '%s'..." % os.path.basename(config_header))

      WriteConfigStatus("openexr_config.status")

      d = os.path.dirname(config_header)
      if not os.path.isdir(d):
         os.makedirs(d)

      with open(config_header, "w") as f:
         if sys.platform == "win32":
            f.write("#define OPENEXR_IMF_HAVE_COMPLETE_IOMANIP 1\n")
         elif sys.platform == "darwin":
            f.write("#define OPENEXR_IMF_HAVE_DARWIN 1\n")
            f.write("#define OPENEXR_IMF_HAVE_COMPLETE_IOMANIP 1\n")
            f.write("#include <string.h>\n")
         else:
            f.write("#define OPENEXR_IMF_HAVE_LINUX_PROCFS 1\n")
            f.write("#define OPENEXR_IMF_HAVE_COMPLETE_IOMANIP 1\n")
            f.write("#define OPENEXR_IMF_HAVE_LARGE_STACK 1\n")
         f.write("\n")

         if namespace_version:
            api_version = "%s_%s" % (lib_version[0], lib_version[1])
            f.write("#define OPENEXR_IMF_INTERNAL_NAMESPACE_CUSTOM 1\n")
            f.write("#define OPENEXR_IMF_NAMESPACE Imf\n")
            f.write("#define OPENEXR_IMF_INTERNAL_NAMESPACE Imf_%s\n" % api_version)
         else:
            f.write("#define OPENEXR_IMF_INTERNAL_NAMESPACE_CUSTOM 0\n")
            f.write("#define OPENEXR_IMF_NAMESPACE Imf\n")
            f.write("#define OPENEXR_IMF_INTERNAL_NAMESPACE Imf\n")
         f.write("\n")

         f.write("#define OPENEXR_VERSION_STRING \"%d.%d.%d\"\n" % lib_version)
         f.write("#define OPENEXR_PACKAGE_STRING \"OpenEXR %d.%d.%d\"\n" % lib_version)
         f.write("#define OPENEXR_VERSION_MAJOR %d\n" % lib_version[0])
         f.write("#define OPENEXR_VERSION_MINOR %d\n" % lib_version[1])
         f.write("#define OPENEXR_VERSION_PATCH %d\n" % lib_version[2])
         f.write("#define OPENEXR_VERSION_HEX ((OPENEXR_VERSION_MAJOR << 24) | \\\n")
         f.write("                             (OPENEXR_VERSION_MINOR << 16) | \\\n")
         f.write("                             (OPENEXR_VERSION_PATCH <<  8))\n")
         f.write("\n")

         if have_gcc_include_asm_avx:
            f.write("#define OPENEXR_IMF_HAVE_GCC_INLINE_ASM_AVX 1\n")

         if have_sysconf_nprocessors_onln:
            f.write("#define OPENEXR_IMF_HAVE_SYSCONF_NPROCESSORS_ONLN 1\n")

def GenerateHeader(target, source, env):
   p = subprocess.Popen([str(source[0])], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
   out, _ = p.communicate()
   with open(str(target[0]), "w") as f:
      for l in out.split("\r\n"):
         f.write(l+"\n")
   return None


# Zlib dependency
def zlibName(static):
  return ("z" if sys.platform != "win32" else ("zlib" if static else "zdll"))

def zlibDefines(static):
  return ([] if (static or sys.platform != "win32") else ["ZLIB_DLL"])

rv = excons.ExternalLibRequire("zlib", libnameFunc=zlibName, definesFunc=zlibDefines)
if rv["require"] is None:
   excons.PrintOnce("OpenEXR: Build zlib from sources ...")
   excons.Call("zlib", targets=["zlib"], imp=["ZlibPath", "RequireZlib"])
   zlibStatic = (excons.GetArgument("zlib-static", 1, int) != 0)
   def zlibRequire(env):
      RequireZlib(env, static=zlibStatic) # pylint: disable=undefined-variable
else:
   zlibRequire = rv["require"]



if sys.platform != "win32":
   env.Append(CPPFLAGS=" -Wno-unused-variable -Wno-unused-parameter")
   if sys.platform == "darwin":
      env.Append(CPPFLAGS=" -Wno-unused-private-field")
   else:
      env.Append(CPPFLAGS=" -Wno-unused-but-set-variable")
else:
   env.Append(CPPDEFINES=["_CRT_SECURE_NO_WARNINGS"])
   # 4127: Conditional expression is constant
   # 4100: Unreferenced format parameter
   env.Append(CPPFLAGS=" /wd4127 /wd4100")

env["BUILDERS"]["GenerateHeader"] = SCons.Script.Builder(action=SCons.Script.Action(GenerateHeader, "Generating $TARGET ..."), suffix=".h")

conf = SCons.Script.Configure(env)
if conf.TryCompile(gcc_include_asm_avx_check_src, ".cpp"):
   have_gcc_include_asm_avx = True
if conf.TryCompile(_sc_nprocessors_onln_check_src, ".cpp"):
   have_sysconf_nprocessors_onln = True
conf.Finish()

binext = ("" if sys.platform != "win32" else ".exe")

eluth = env.GenerateHeader("IlmBase/Half/eLut.h", SCons.Script.File("%s/bin/generators/eLut%s" % (excons.OutputBaseDirectory(), binext)))
tofloath = env.GenerateHeader("IlmBase/Half/toFloat.h", SCons.Script.File("%s/bin/generators/toFloat%s" % (excons.OutputBaseDirectory(), binext)))
b44h = env.GenerateHeader("OpenEXR/IlmImf/b44ExpLogTable.h", SCons.Script.File("%s/bin/generators/b44ExpLogTable%s" % (excons.OutputBaseDirectory(), binext)))
dwah = env.GenerateHeader("OpenEXR/IlmImf/dwaLookups.h", SCons.Script.File("%s/bin/generators/dwaLookups%s" % (excons.OutputBaseDirectory(), binext)))

out_headers_dir = "%s/include/OpenEXR" % excons.OutputBaseDirectory()

GenerateIlmBaseConfig("%s/IlmBaseConfig.h" % out_headers_dir)

GenerateOpenEXRConfig("%s/OpenEXRConfig.h" % out_headers_dir)

half_headers = env.Install(out_headers_dir, ["IlmBase/Half/half.h",
                                             "IlmBase/Half/halfExport.h",
                                             "IlmBase/Half/halfFunction.h",
                                             "IlmBase/Half/halfLimits.h"])

iex_headers = env.Install(out_headers_dir, excons.glob("IlmBase/Iex/*.h"))

iexmath_headers = env.Install(out_headers_dir, excons.glob("IlmBase/IexMath/*.h"))

imath_headers = env.Install(out_headers_dir, excons.glob("IlmBase/Imath/*.h"))

ilmthread_headers = env.Install(out_headers_dir, excons.glob("IlmBase/IlmThread/*.h"))

ilmthread_srcs = excons.glob("IlmBase/IlmThread/*.cpp")
if sys.platform != "win32":
   ilmthread_srcs = filter(lambda x: "Win32" not in x, ilmthread_srcs)


nowarn_flags = ""
if sys.platform != "win32":
   nowarn_flags += " -Wno-sign-compare -Wno-unused-function -Wno-switch"
   if sys.platform == "darwin":
      nowarn_flags += " -Wno-unused-local-typedef -Wno-missing-field-initializers -Wno-tautological-compare"
   else:
      nowarn_flags += " -Wno-unused-local-typedefs -Wno-maybe-uninitialized -Wno-type-limits -Wno-return-type -Wno-extra"

openexr_defs = []
if zlib_win_api:
   openexr_defs.append("ZLIB_WINAPI")

def ilmimf_filter(x):
   name = os.path.splitext(os.path.basename(x))[0]
   return (name not in ["b44ExpLogTable", "dwaLookups"])

ilmimf_headers = env.Install(out_headers_dir, filter(ilmimf_filter, excons.glob("OpenEXR/IlmImf/*.h")))

ilmimf_srcs = filter(ilmimf_filter, excons.glob("OpenEXR/IlmImf/*.cpp"))

ilmimfutil_headers = env.Install(out_headers_dir, excons.glob("OpenEXR/IlmImfUtil/*.h"))

pyiex_headers = env.Install(out_headers_dir, excons.glob("PyIlmBase/PyIex/*.h"))

def pyiex_filter(x):
   name = os.path.splitext(os.path.basename(x))[0]
   return (name not in ["iexmodule"])

pyiex_all_srcs = excons.glob("PyIlmBase/PyIex/*.cpp")

pyiex_srcs = filter(pyiex_filter, pyiex_all_srcs)

pyimath_headers = env.Install(out_headers_dir, excons.glob("PyIlmBase/PyImath/*.h"))

def pyimath_filter(x):
   name = os.path.splitext(os.path.basename(x))[0]
   return (name not in ["imathmodule", "PyImathM44Array"])

pyimath_all_srcs = excons.glob("PyIlmBase/PyImath/*.cpp")

pyimath_srcs = filter(pyimath_filter, pyimath_all_srcs)

py_defs = []
if sys.platform != "win32":
   py_defs.append("PLATFORM_VISIBILITY_AVAILABLE")
   py_defs.append("BOOST_PYTHON_USE_GCC_SYMBOL_VISIBILITY")

pymod_defs = []
if pyilmbase_static:
   pymod_defs.append("PYILMBASE_STATICLIBS")
if excons.GetArgument("boost-python-static", excons.GetArgument("boost-static", 0, int), int) != 0:
   pymod_defs.append("PYILMBASE_USE_STATIC_BOOST_PYTHON")

prjs = []

ilmbase_incdirs = ["IlmBase/Half", "IlmBase/Iex", "IlmBase/IexMath", "IlmBase/Imath", "IlmBase/IlmThread"]
openexr_incdirs = ["OpenEXR/IlmImf", "OpenEXR/IlmImfUtil"]
python_incdirs  = ["PyIlmBase/PyIex", "PyIlmBase/PyImath"]
configs_incdirs = [out_headers_dir]


# Half

def HalfName(static=False):
  name = "Half" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def HalfPath(static=False):
  name = HalfName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireHalf(env, static=False):
  if not static and sys.platform == "win32":
    env.Append(CPPDEFINES=["OPENEXR_DLL"])
  env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include",
                      excons.OutputBaseDirectory() + "/include/OpenEXR"])
  excons.Link(env, HalfPath(static), static=static, force=True, silent=True)

prjs.append({"name": "eLut",
             "type": "program",
             "desc": "Half library header generator",
             "prefix": "generators",
             "srcs": ["IlmBase/Half/eLut.cpp"]})

prjs.append({"name": "toFloat",
             "type": "program",
             "desc": "Half library header generator",
             "prefix": "generators",
             "srcs": ["IlmBase/Half/toFloat.cpp"]})

prjs.append({"name": HalfName(True),
             "type": "staticlib",
             "alias": "Half-static",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": ["IlmBase/Half/half.cpp"]})

prjs.append({"name": HalfName(False),
             "type": "sharedlib",
             "alias": "Half-shared",
             "defs": (["OPENEXR_DLL", "HALF_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": ["IlmBase/Half/half.cpp"]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libHalf.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libHalf.%d.dylib" % lib_version[0]

# Iex

def IexName(static=False):
  name = "Iex" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def IexPath(static=False):
  name = IexName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname

prjs.append({"name": IexName(True),
             "type": "staticlib",
             "alias": "Iex-static",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/Iex/*.cpp")})

prjs.append({"name": IexName(False),
             "type": "sharedlib",
             "alias": "Iex-shared",
             "defs": (["OPENEXR_DLL", "IEX_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/Iex/*.cpp")})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libIex.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libIex.%d.dylib" % lib_version[0]

# IexMath

def IexMathName(static=False):
  name = "IexMath" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def IexMathPath(static=False):
  name = IexMathName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname

prjs.append({"name": IexMathName(True),
             "type": "staticlib",
             "alias": "IexMath-static",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/IexMath/*.cpp")})

prjs.append({"name": IexMathName(False),
             "type": "sharedlib",
             "alias": "IexMath-shared",
             "defs": (["OPENEXR_DLL", "IEXMATH_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/IexMath/*.cpp"),
             "libs": [SCons.Script.File(IexPath(False))]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libIexMath.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libIexMath.%d.dylib" % lib_version[0]

# Imath

def ImathName(static=False):
  name = "Imath" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def ImathPath(static=False):
  name = ImathName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireImath(env, static=False):
  if not static and sys.platform == "win32":
    env.Append(CPPDEFINES=["OPENEXR_DLL"])
  env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include",
                      excons.OutputBaseDirectory() + "/include/OpenEXR"])
  excons.Link(env, ImathPath(static), static=static, force=True, silent=True)
  excons.Link(env, IexMathPath(static), static=static, force=True, silent=True)
  excons.Link(env, IexPath(static), static=static, force=True, silent=True)
  excons.Link(env, HalfPath(static), static=static, force=True, silent=True)


prjs.append({"name": ImathName(True),
             "type": "staticlib",
             "alias": "Imath-static",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/Imath/*.cpp")})

prjs.append({"name": ImathName(False),
             "type": "sharedlib",
             "alias": "Imath-shared",
             "defs": (["OPENEXR_DLL", "IMATH_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/Imath/*.cpp"),
             "libs": [SCons.Script.File(IexPath(False))]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libImath.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libImath.%d.dylib" % lib_version[0]

# IlmThread

def IlmThreadName(static=False):
  name = "IlmThread" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def IlmThreadPath(static=False):
  name = IlmThreadName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireIlmThread(env, static=False):
  if not static and sys.platform == "win32":
    env.Append(CPPDEFINES=["OPENEXR_DLL"])
  env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include",
                      excons.OutputBaseDirectory() + "/include/OpenEXR"])
  excons.Link(env, IlmThreadPath(static), static=static, force=True, silent=True)
  excons.Link(env, IexPath(static), static=static, force=True, silent=True)

prjs.append({"name": IlmThreadName(True),
             "type": "staticlib",
             "alias": "IlmThread-static",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": ilmthread_srcs})

prjs.append({"name": IlmThreadName(False),
             "type": "sharedlib",
             "alias": "IlmThread-shared",
             "defs": (["OPENEXR_DLL", "ILMTHREAD_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + configs_incdirs,
             "srcs": ilmthread_srcs,
             "libs": [SCons.Script.File(IexPath(False))]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libIlmThread.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libIlmThread.%d.dylib" % lib_version[0]

# IlmImf

def IlmImfName(static=False):
  name = "IlmImf" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def IlmImfPath(static=False):
  name = IlmImfName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireIlmImf(env, static=False):
  if not static and sys.platform == "win32":
    env.Append(CPPDEFINES=["OPENEXR_DLL"])
  env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include",
                      excons.OutputBaseDirectory() + "/include/OpenEXR"])
  excons.Link(env, IlmImfPath(static), static=static, force=True, silent=True)
  excons.Link(env, IlmThreadPath(static), static=static, force=True, silent=True)
  excons.Link(env, ImathPath(static), static=static, force=True, silent=True)
  excons.Link(env, IexMathPath(static), static=static, force=True, silent=True)
  excons.Link(env, IexPath(static), static=static, force=True, silent=True)
  excons.Link(env, HalfPath(static), static=static, force=True, silent=True)

prjs.append({"name": "b44ExpLogTable",
             "type": "program",
             "desc": "IlmImf library header generator",
             "prefix": "generators",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": ["OpenEXR/IlmImf/b44ExpLogTable.cpp"],
             "libs": [SCons.Script.File(IlmThreadPath(True)),
                      SCons.Script.File(IexPath(True)),
                      SCons.Script.File(HalfPath(True))],
             "custom": [threads.Require]})

prjs.append({"name": "dwaLookups",
             "type": "program",
             "prefix": "generators",
             "desc": "IlmImf library header generator",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": ["OpenEXR/IlmImf/dwaLookups.cpp"],
             "libs": [SCons.Script.File(IlmThreadPath(True)),
                      SCons.Script.File(IexPath(True)),
                      SCons.Script.File(HalfPath(True))],
             "custom": [threads.Require]})

prjs.append({"name": IlmImfName(True),
             "type": "staticlib",
             "alias": "IlmImf-static",
             "symvis": "default",
             "defs": openexr_defs,
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": ilmimf_srcs,
             "custom": [zlibRequire]})

prjs.append({"name": IlmImfName(False),
             "type": "sharedlib",
             "alias": "IlmImf-shared",
             "defs": openexr_defs + (["OPENEXR_DLL", "ILMIMF_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": ilmimf_srcs,
             "libs": [SCons.Script.File(IlmThreadPath(False)),
                      SCons.Script.File(ImathPath(False)),
                      SCons.Script.File(IexPath(False)),
                      SCons.Script.File(HalfPath(False))],
             "custom": [threads.Require, zlibRequire]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libIlmImf.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libIlmImf.%d.dylib" % lib_version[0]

# IlmImfUtil

def IlmImfUtilName(static=False):
  name = "IlmImfUtil" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def IlmImfUtilPath(static=False):
  name = IlmImfUtilName(static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/" + libname


prjs.append({"name": IlmImfUtilName(True),
             "type": "staticlib",
             "alias": "IlmImfUtil-static",
             "symvis": "default",
             "defs": openexr_defs,
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": excons.glob("OpenEXR/IlmImfUtil/*.cpp"),
             "custom": [zlibRequire]})

prjs.append({"name": IlmImfUtilName(False),
             "type": "sharedlib",
             "alias": "IlmImfUtil-shared",
             "defs": openexr_defs + (["OPENEXR_DLL", "ILMIMF_EXPORTS"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": excons.glob("OpenEXR/IlmImfUtil/*.cpp"),
             "libs": [SCons.Script.File(IlmImfPath(False)),
                      SCons.Script.File(IlmThreadPath(False)),
                      SCons.Script.File(ImathPath(False)),
                      SCons.Script.File(IexPath(False)),
                      SCons.Script.File(HalfPath(False))],
             "custom": [threads.Require, zlibRequire]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libIlmImfUtil.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libIlmImfUtil.%d.dylib" % lib_version[0]

# Python

def PyIexName(static=False):
  name = "PyIex" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def PyIexPath(static=False):
  name = PyIexName(static=static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/python/" + python.Version() + "/" + libname

def RequirePyIex(env, staticpy=False, staticbase=False):
  if not staticbase and sys.platform == "win32":
    env.Append(CPPDEFINES=["OPENEXR_DLL"])
  if staticpy:
    env.Append(CPPDEFINES=["PYILMBASE_STATICLIBS"])
    if sys.platform == "win32":
      env.Append(CPPDEFINES=["PLATFORM_BUILD_STATIC"])
  if sys.platform != "win32":
    env.Append(CPPDEFINES=["PLATFORM_VISIBILITY_AVAILABLE"])
  else:
    env.Append(CPPDEFINES=["PLATFORM_WINDOWS"])
  excons.Link(env, PyIexPath(staticpy), static=staticpy, force=True, silent=True)
  if staticpy:
    excons.Link(env, IexMathPath(staticbase), static=staticbase, force=True, silent=True)
    excons.Link(env, IexPath(staticbase), static=staticbase, force=True, silent=True)
    boost.Require(libs=["python"])
  python.SoftRequire(env)

def PyImathName(static=False):
  name = "PyImath" + lib_suffix
  if sys.platform == "win32" and static:
    name = "lib" + name
  return name

def PyImathPath(static=False):
  name = PyImathName(static=static)
  if sys.platform == "win32":
    libname = name + ".lib"
  else:
    libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
  return excons.OutputBaseDirectory() + "/lib/python/" + python.Version() + "/" + libname

def RequirePyImath(env, staticpy=False, staticbase=False):
  if staticpy:
    env.Append(CPPDEFINES=["PYILMBASE_STATICLIBS"])
    if sys.platform == "win32":
      env.Append(CPPDEFINES=["PLATFORM_BUILD_STATIC"])
  if sys.platform != "win32":
    env.Append(CPPDEFINES=["PLATFORM_VISIBILITY_AVAILABLE"])
  else:
    env.Append(CPPDEFINES=["PLATFORM_WINDOWS"])
  excons.Link(env, PyImathPath(staticpy), static=staticpy, force=True, silent=True)
  excons.Link(env, PyIexPath(staticpy), static=staticpy, force=True, silent=True)
  if staticpy:
    RequireImath(env, static=staticbase)
    boost.Require(libs=["python"])(env)
  python.SoftRequire(env)

prjs.append({"name": PyIexName(True),
             "type": "staticlib",
             "desc": "Iex python helper library",
             "symvis": "default",
             "alias": "PyIex-static",
             "prefix": "python/" + python.Version(),
             "bldprefix": "python" + python.Version(),
             "defs": py_defs + ["PYIEX_EXPORTS"] + (["PLATFORM_BUILD_STATIC"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": pyiex_srcs,
             "custom": [python.SoftRequire, boost.Require(libs=["python"])]})

prjs.append({"name": PyIexName(False),
             "type": "sharedlib",
             "desc": "Iex python helper library",
             "alias": "PyIex-shared",
             "win_separate_dll_and_lib": False,
             "prefix": "python/" + python.Version(),
             "bldprefix": "python" + python.Version(),
             "defs": ["PYIEX_EXPORTS"] + py_defs + (["OPENEXR_DLL"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": pyiex_srcs,
             "libs": [SCons.Script.File(IexMathPath(False)),
                      SCons.Script.File(IexPath(False))],
             "custom": [python.SoftRequire, boost.Require(libs=["python"])]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libPyIex.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libPyIex.%d.dylib" % lib_version[0]

prjs.append({"name": PyImathName(True),
             "type": "staticlib",
             "desc": "Imath python helper library",
             "symvis": "default",
             "alias": "PyImath-static",
             "prefix": "python/" + python.Version(),
             "bldprefix": "python" + python.Version(),
             "defs": py_defs + ["PYIMATH_EXPORTS"] + (["PLATFORM_BUILD_STATIC"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": [out_headers_dir],
             "srcs": pyimath_srcs,
             "custom": [python.SoftRequire, boost.Require(libs=["python"])]})

prjs.append({"name": PyImathName(False),
             "type": "sharedlib",
             "desc": "Imath python helper library",
             "symvis": "default",
             "alias": "PyImath-shared",
             "win_separate_dll_and_lib": False,
             "prefix": "python/" + python.Version(),
             "bldprefix": "python" + python.Version(),
             "defs": ["PYIMATH_EXPORTS"] + py_defs + (["OPENEXR_DLL"] if sys.platform == "win32" else []),
             "cppflags": nowarn_flags,
             "incdirs": [out_headers_dir],
             "srcs": pyimath_srcs,
             "libs": [SCons.Script.File(PyIexPath(False)),
                      SCons.Script.File(ImathPath(False)),
                      SCons.Script.File(IexMathPath(False)),
                      SCons.Script.File(IexPath(False))],
             "custom": [python.SoftRequire, boost.Require(libs=["python"])]})

if not lib_suffix:
   prjs[-1]["version"] = lib_version_str
   prjs[-1]["soname"] = "libPyImath.so.%d" % lib_version[0]
   prjs[-1]["install_name"] = "libPyImath.%d.dylib" % lib_version[0]

iexmodulename = ("iex" if sys.platform == "win32" else "iexmodule")

prjs.append({"name": iexmodulename,
             "type": "dynamicmodule",
             "alias": "Iex-python",
             "desc": "Iex library python bindings",
             "ext": python.ModuleExtension(),
             "prefix": python.ModulePrefix() + "/" + python.Version(),
             "bldprefix": "python" + python.Version(),
             "defs": py_defs + pymod_defs,
             "cppflags": nowarn_flags,
             "incdirs": [out_headers_dir],
             "srcs": ["PyIlmBase/PyIex/iexmodule.cpp"],
             "libs": [SCons.Script.File(PyIexPath(pyilmbase_static))] +
                     [SCons.Script.File(IexMathPath(True)), SCons.Script.File(IexPath(True))] if pyilmbase_static else [],
             "custom": [python.SoftRequire, boost.Require(libs=["python"])]})

imathmodulename = ("imath" if sys.platform == "win32" else "imathmodule")

prjs.append({"name": imathmodulename,
             "type": "dynamicmodule",
             "alias": "Imath-python",
             "desc": "Imath library python bindings",
             "ext": python.ModuleExtension(),
             "prefix": python.ModulePrefix() + "/" + python.Version(),
             "bldprefix": "python" + python.Version(),
             "defs": py_defs + pymod_defs,
             "cppflags": nowarn_flags,
             "incdirs": [out_headers_dir],
             "srcs": ["PyIlmBase/PyImath/imathmodule.cpp"],
             "libs": [SCons.Script.File(PyImathPath(pyilmbase_static)), SCons.Script.File(PyIexPath(pyilmbase_static))] +
                     [SCons.Script.File(IexMathPath(True)), SCons.Script.File(ImathPath(True)), SCons.Script.File(IexPath(True))] if pyilmbase_static else [],
             "custom": [python.SoftRequire, boost.Require(libs=["python"])]})

# Command line tools
for f in excons.glob("OpenEXR/exr*/CMakeLists.txt"):
   d = os.path.dirname(f)
   prjs.append({"name": os.path.basename(d),
                "type": "program",
                "desc": "Command line tool",
                "symvis": "default",
                "defs": openexr_defs,
                "cppflags": nowarn_flags,
                "incdirs": [d] + ilmbase_incdirs + openexr_incdirs + configs_incdirs,
                "srcs": excons.glob(d+"/*.cpp"),
                "libs": [SCons.Script.File(IlmImfPath(True)),
                         SCons.Script.File(IlmThreadPath(True)),
                         SCons.Script.File(ImathPath(True)),
                         SCons.Script.File(IexPath(True)),
                         SCons.Script.File(HalfPath(True))],
                "custom": [threads.Require, zlibRequire]})

# Tests
prjs.append({"name": "HalfTest",
             "type": "program",
             "desc": "Half library tests",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ["IlmBase/HalfTest"] + ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/HalfTest/*.cpp"),
             "libs": [SCons.Script.File(HalfPath(True))]})

prjs.append({"name": "IexTest",
             "type": "program",
             "desc": "Iex library tests",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ["IlmBase/IexTest"] + ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/IexTest/*.cpp"),
             "libs": [SCons.Script.File(IexPath(True))]})

prjs.append({"name": "ImathTest",
             "type": "program",
             "desc": "Imath library tests",
             "symvis": "default",
             "cppflags": nowarn_flags,
             "incdirs": ["IlmBase/ImathTest"] + ilmbase_incdirs + configs_incdirs,
             "srcs": excons.glob("IlmBase/ImathTest/*.cpp"),
             "libs": [SCons.Script.File(ImathPath(True)),
                      SCons.Script.File(IexPath(True))]})

prjs.append({"name": "IlmImfTest",
             "type": "program",
             "desc": "IlmImf library tests",
             "symvis": "default",
             "defs": openexr_defs,
             "cppflags": nowarn_flags,
             "incdirs": ["OpenEXR/IlmImfTest"] + ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": excons.glob("OpenEXR/IlmImfTest/*.cpp"),
             "libs": [SCons.Script.File(IlmImfPath(True)),
                      SCons.Script.File(IlmThreadPath(True)),
                      SCons.Script.File(ImathPath(True)),
                      SCons.Script.File(IexPath(True)),
                      SCons.Script.File(HalfPath(True))],
             "custom": [threads.Require, zlibRequire]})

prjs.append({"name": "IlmImfUtilTest",
             "type": "program",
             "desc": "IlmImfUtil library tests",
             "symvis": "default",
             "defs": openexr_defs,
             "cppflags": nowarn_flags,
             "incdirs": ["OpenEXR/IlmImfUtilTest"] + ilmbase_incdirs + openexr_incdirs + configs_incdirs,
             "srcs": excons.glob("OpenEXR/IlmImfUtilTest/*.cpp"),
             "libs": [SCons.Script.File(IlmImfUtilPath(True)),
                      SCons.Script.File(IlmImfPath(True)),
                      SCons.Script.File(IlmThreadPath(True)),
                      SCons.Script.File(ImathPath(True)),
                      SCons.Script.File(IexPath(True)),
                      SCons.Script.File(HalfPath(True))],
             "custom": [threads.Require, zlibRequire]})

prjs.append({"name": "PyIlmBaseTest",
             "type": "install",
             "desc": "PyIlmBase tests",
             "install": {"lib/python/%s" % python.Version(): ["PyIlmBase/PyIexTest/pyIexTest.py",
                                                              "PyIlmBase/PyImathTest/pyImathTest.py"]}})

# Help setup (scons -h)

excons.AddHelpOptions(openexr="""OPENEXR OPTIONS
  openexr-suffix=<str>          : Library suffix                     ["-2_2"]
  openexr-namespace-version=0|1 : Internally use versioned namespace [1]
  ilmbase-python-staticlibs=0|1 : Link PyIex and PyImath static libs [1]
                                  for iex and imath python modules
  openexr-zlib-winapi=0|1       : Use zlib win API                   [0]""")

targets_help = {"Half-static": "Half static library",
                "Half-shared": "Half shared library",
                "Iex-static": "Iex static library",
                "Iex-shared": "Iex shared library",
                "IexMath-static": "IexMath static library",
                "IexMath-shared": "IexMath shared library",
                "Imath-static": "Imath static library",
                "Imath-shared": "Imath shared library",
                "IlmThread-static": "IlmThread static library",
                "IlmThread-shared": "IlmThread shared library",
                "IlmImf-static": "IlmImf static library",
                "IlmImf-shared": "IlmImf shared library",
                "IlmImfUtil-static": "IlmImfUtil static library",
                "IlmImfUtil-shared": "IlmImfUtil shared library",
                "PyIex-static": "Iex python static library",
                "PyIex-shared": "Iex python shared library",
                "PyImath-static": "Imath python static library",
                "PyImath-shared": "Imath python shared library",
                "openexr": "All libraries",
                "openexr-static": "All static libraries",
                "openexr-shared": "All shared libraries",
                "ilmbase": "All IlmBase libraries",
                "ilmbase-static": "All IlmBase static libraries",
                "ilmbase-shared": "All IlmBase shared librarues",
                "openexr-tools": "All command line tools",
                "ilmbase-python": "All python bindings",
                "openexr-tests": "All tests"}

sameHalfName = (HalfName(True) == HalfName(False))
if sameHalfName:
  targets_help[HalfName(True)] = "Half static and shared libraries"
if lib_suffix or not sameHalfName:
  targets_help["Half"] = "Half static and shared libraries"

sameIexName = (IexName(True) == IexName(False))
if sameIexName:
  targets_help[IexName(True)] = "Iex static and shared libraries"
if lib_suffix or not sameIexName:
  targets_help["Iex"] = "Iex static and shared libraries"

sameIexMathName = (IexMathName(True) == IexMathName(False))
if sameIexMathName:
  targets_help[IexMathName(True)] = "IexMath static and shared libraries"
if lib_suffix or not sameIexMathName:
  targets_help["IexMath"] = "IexMath static and shared libraries"

sameImathName = (ImathName(True) == ImathName(False))
if sameImathName:
  targets_help[ImathName(True)] = "Imath static and shared libraries"
if lib_suffix or not sameImathName:
  targets_help["Imath"] = "Imath static and shared libraries"

sameIlmThreadName = (IlmThreadName(True) == IlmThreadName(False))
if sameIlmThreadName:
  targets_help[IlmThreadName(True)] = "IlmThread static and shared libraries"
if lib_suffix or not sameIlmThreadName:
  targets_help["IlmThread"] = "IlmThread static and shared libraries"

sameIlmImfName = (IlmImfName(True) == IlmImfName(False))
if sameIlmImfName:
  targets_help[IlmImfName(True)] = "IlmImf static and shared libraries"
if lib_suffix or not sameIlmImfName:
  targets_help["IlmImf"] = "IlmImf static and shared libraries"

sameIlmImfUtilName = (IlmImfUtilName(True) == IlmImfUtilName(False))
if sameIlmImfUtilName:
  targets_help[IlmImfUtilName(True)] = "IlmImfUtil static and shared libraries"
if lib_suffix or not sameIlmImfUtilName:
  targets_help["IlmImfUtil"] = "IlmImfUtil static and shared libraries"

samePyIexName = (PyIexName(True) == PyIexName(False))
if samePyIexName:
  targets_help[PyIexName(True)] = "Iex python static and shared libraries"
if lib_suffix or not samePyIexName:
  targets_help["PyIex"] = "Iex python static and shared libraries"

samePyImathName = (PyImathName(True) == PyImathName(False))
if samePyImathName:
  targets_help[PyImathName(True)] = "Imath python static and shared libraries"
if lib_suffix or not samePyImathName:
  targets_help["PyImath"] = "Imath python static and shared libraries"

excons.AddHelpTargets(targets_help)

tgts = excons.DeclareTargets(env, prjs)

env.Depends(tgts["Half-static"], half_headers)
env.Depends(tgts["Half-shared"], half_headers)
if lib_suffix or not sameHalfName:
  env.Alias("Half", ["Half-static", "Half-shared"])

env.Depends(tgts["Iex-static"], iex_headers)
env.Depends(tgts["Iex-shared"], iex_headers)
if lib_suffix or not sameIexName:
  env.Alias("Iex", ["Iex-static", "Iex-shared"])

env.Depends(tgts["IexMath-static"], iexmath_headers)
env.Depends(tgts["IexMath-shared"], iexmath_headers)
if lib_suffix or not sameIexMathName:
  env.Alias("IexMath", ["IexMath-static", "IexMath-shared"])

env.Depends(tgts["Imath-static"], imath_headers)
env.Depends(tgts["Imath-shared"], imath_headers)
if lib_suffix or not sameImathName:
  env.Alias("Imath", ["Imath-static", "Imath-shared"])

env.Depends(tgts["IlmThread-static"], ilmthread_headers)
env.Depends(tgts["IlmThread-shared"], ilmthread_headers)
if lib_suffix or not sameIlmThreadName:
  env.Alias("IlmThread", ["IlmThread-static", "IlmThread-shared"])

env.Depends(tgts["IlmImf-static"], ilmimf_headers)
env.Depends(tgts["IlmImf-shared"], ilmimf_headers)
if lib_suffix or not sameIlmImfName:
  env.Alias("IlmImf", ["IlmImf-static", "IlmImf-shared"])

env.Depends(tgts["IlmImfUtil-static"], ilmimfutil_headers)
env.Depends(tgts["IlmImfUtil-shared"], ilmimfutil_headers)
if lib_suffix or not sameIlmImfUtilName:
  env.Alias("IlmImfUtil", ["IlmImfUtil-static", "IlmImfUtil-shared"])

if lib_suffix or not samePyIexName:
  env.Alias("PyIex", ["PyIex-static", "PyIex-shared"])

if lib_suffix or not samePyImathName:
  env.Alias("PyImath", ["PyImath-static", "PyImath-shared"])

env.Alias("openexr-static", [tgts["Half-static"],
                             tgts["Iex-static"],
                             tgts["IexMath-static"],
                             tgts["Imath-static"],
                             tgts["IlmThread-static"],
                             tgts["IlmImf-static"],
                             tgts["IlmImfUtil-static"]])

env.Alias("openexr-shared", [tgts["Half-shared"],
                             tgts["Iex-shared"],
                             tgts["IexMath-shared"],
                             tgts["Imath-shared"],
                             tgts["IlmThread-shared"],
                             tgts["IlmImf-shared"],
                             tgts["IlmImfUtil-shared"]])

env.Alias("ilmbase-static", [tgts["Half-static"],
                             tgts["Iex-static"],
                             tgts["IexMath-static"],
                             tgts["Imath-static"],
                             tgts["IlmThread-static"]])

env.Alias("ilmbase-shared", [tgts["Half-shared"],
                             tgts["Iex-shared"],
                             tgts["IexMath-shared"],
                             tgts["Imath-shared"],
                             tgts["IlmThread-shared"]])

env.Alias("ilmbase", ["ilmbase-static", "ilmbase-shared"])

env.Alias("openexr-tools", [tgts[y] for y in filter(lambda x: x.startswith("exr"), tgts.keys())])

env.Alias("ilmbase-python", [tgts["PyIex-static"],
                             tgts["PyIex-shared"],
                             tgts["PyImath-static"],
                             tgts["PyImath-shared"],
                             tgts[iexmodulename],
                             tgts[imathmodulename]])

env.Alias("openexr", ["openexr-static", "openexr-shared", "ilmbase-python", "openexr-tools"])

env.Alias("openexr-tests", [tgts["HalfTest"],
                            tgts["IexTest"],
                            tgts["ImathTest"],
                            tgts["IlmImfTest"],
                            tgts["IlmImfUtilTest"],
                            tgts["PyIlmBaseTest"]])

SCons.Script.Export("HalfName HalfPath RequireHalf IexName IexPath IexMathName IexMathPath ImathName ImathPath RequireImath IlmThreadName IlmThreadPath RequireIlmThread IlmImfName IlmImfPath RequireIlmImf IlmImfUtilName IlmImfUtilPath PyIexName PyIexPath RequirePyIex PyImathName PyImathPath RequirePyImath")
