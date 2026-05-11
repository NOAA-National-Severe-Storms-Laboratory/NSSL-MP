.SUFFIXES: .F90 .o
# Makefile used for MPAS
all: dummy microphysics

dummy:
	echo "****** compiling NSSL microphysics ******"

OBJS = \
	mp_nssl.o    \
	module_mp_nssl_2mom.o

microphysics: $(OBJS)
	ar -ru ./../../libphys.a $(OBJS)

# DEPENDENCIES:
mp_nssl.o: module_mp_nssl_2mom.o
#mp_nssl.o: module_mp_nssl_2mom.o mp_nssl.F90
#	$(FC) $(CPPFLAGS) $(COREDEF) $(FFLAGS) -c $*.F90 $(CPPINCLUDES) $(FCINCLUDES) -I../.. -I../../physics_wrf -I../../physics_mmm -I../../../../framework -I../../../../external/esmf_time_f90

#module_mp_nssl_2mom.o: module_mp_nssl_2mom.F90
#	$(FC) $(CPPFLAGS) $(COREDEF) $(FFLAGS) -c $*.F90 $(CPPINCLUDES) $(FCINCLUDES) -I../.. -I../../physics_wrf -I../../physics_mmm -I../../../../framework -I../../../../external/esmf_time_f90
clean:
	$(RM) *.o *.mod
	@# Certain systems with intel compilers generate *.i files
	@# This removes them during the clean process
	$(RM) *.i

%.o %.mod: %.F90
	$(FC) $(CPPFLAGS) $(COREDEF) $(FFLAGS) -c $*.F90 $(CPPINCLUDES) $(FCINCLUDES) -I../.. -I../../physics_wrf -I../../physics_mmm -I../../../../framework -I../../../../external/esmf_time_f90
