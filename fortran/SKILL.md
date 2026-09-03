---
name: fortran
description: "Best practices for modern Fortran (2003/2008+) scientific and numerical computing, covering modules, explicit interfaces, kind parameters, memory safety, and testing. Use when writing or reviewing Fortran source (.f90/.f95/.f03/.f08), defining modules and derived types, choosing numeric kind parameters, working with allocatable arrays, setting up a Fortran build with CMake or fpm, or writing unit tests for numerical code."
---

# Modern Fortran Development

This skill covers writing modern, maintainable Fortran (2003/2008 and later) for scientific and numerical computing, including module organization, kind parameters, procedure design, memory safety, and testing.

## Workflow for Writing a Modern Fortran Module

1. **Define shared kinds first** — Create a `kinds_mod` (or similarly named) module with `iso_fortran_env` or `selected_real_kind`/`selected_int_kind` parameters used across the whole project.
2. **Design the module** — Group related derived types and procedures into one focused module per file; declare `implicit none` at the top.
3. **Write procedure interfaces** — Give every dummy argument an explicit `intent(in)`, `intent(out)`, or `intent(inout)`; use `use, only:` to import exactly what's needed.
4. **Implement with early validation** — Check preconditions (array bounds, allocation state, valid ranges) at the top of each procedure and return/stop early rather than nesting deeply.
5. **Manage arrays explicitly** — Use allocatable arrays, check `allocated()` before use, and deallocate when the lifetime isn't naturally scoped.
6. **Build with warnings on** — Compile with `-Wall -Wextra -std=f2008` (gfortran) or the equivalent, and fail CI on new warnings.
7. **Test** — Write unit tests for individual procedures and integration tests for full numerical workflows, checking tolerances rather than exact floating-point equality.

## Basic Principles

- Target modern Fortran standards — Fortran 2003, 2008, or newer — and avoid writing in a legacy FORTRAN 77 style just because the compiler still accepts it.
- Put `implicit none` in every program unit (module, program, and — via inheritance from a module or explicit statement — every procedure) so undeclared-variable typos are caught at compile time instead of producing silent wrong answers.
- Put procedures in modules rather than external subprograms; module procedures get automatically-generated explicit interfaces, which lets the compiler catch argument mismatches that external procedures cannot.
- Keep modules focused on one concern and place each major module in its own file, named to match the module (e.g., `module linear_solver` in `linear_solver.f90`).
- Prefer clear, structured code over clever language tricks — Fortran numerical code is read far more often than it's written, usually by someone other than the original author.
- Avoid obsolete features: `COMMON` blocks (replace with modules), `GOTO`-heavy control flow (replace with structured `if`/`do`/`select case`, plus `block` where useful), and numeric statement labels used as jump targets.

## Kinds and Types

- Define numeric kind parameters in one shared module (e.g., `kind_mod` or `precision_mod`) so the whole codebase can change precision in one place.
- Use `real(kind=dp)` (or the project's approved real kind) for floating-point values — never bare `real` or `double precision`, whose actual precision is compiler- and flag-dependent.
- Use `integer(kind=i4)` (or the project's approved integer kind) for integers where the width matters, especially in interfaces to C or binary I/O.
- Define constants such as `pi` explicitly and precisely (e.g., `real(dp), parameter :: pi = 4.0_dp * atan(1.0_dp)`) rather than truncated literals.
- Include the physical units in a comment for any variable representing a physical quantity (`real(dp) :: velocity  ! m/s`).
- Use derived types to group related data (e.g., a `particle_t` type with position, velocity, and mass fields) instead of passing many loose primitive arguments through procedure calls.

### Example: Kinds Module and a Numerical Procedure

```fortran
module kinds_mod
  use iso_fortran_env, only: real64, int32
  implicit none
  private
  public :: dp, i4

  integer, parameter :: dp = real64
  integer, parameter :: i4 = int32
end module kinds_mod
```

```fortran
module stats_mod
  use kinds_mod, only: dp
  implicit none
  private
  public :: mean, standard_deviation

contains

  pure function mean(x) result(m)
    real(dp), intent(in) :: x(:)
    real(dp) :: m

    m = sum(x) / real(size(x), dp)
  end function mean

  pure function standard_deviation(x) result(s)
    real(dp), intent(in) :: x(:)
    real(dp) :: s
    real(dp) :: m
    integer :: n

    n = size(x)
    if (n < 2) then
      s = 0.0_dp
      return
    end if

    m = mean(x)
    s = sqrt(sum((x - m)**2) / real(n - 1, dp))
  end function standard_deviation

end module stats_mod
```

```fortran
program demo
  use kinds_mod, only: dp
  use stats_mod, only: mean, standard_deviation
  implicit none

  real(dp) :: samples(5)
  samples = [1.0_dp, 2.0_dp, 3.0_dp, 4.0_dp, 5.0_dp]

  print '(A, F0.4)', 'mean = ', mean(samples)
  print '(A, F0.4)', 'stddev = ', standard_deviation(samples)
end program demo
```

## Naming and Style

- Use lowercase for language keywords and most identifiers; Fortran is case-insensitive, so consistent lowercase avoids visual noise from mixed-case keywords.
- Use underscores for multi-word names (`particle_velocity`, not `particleVelocity` or `ParticleVelocity`).
- Avoid names that differ only by case (`Count` vs `count`) since Fortran treats them as identical, inviting confusion.
- Use descriptive names for procedures and state (`compute_residual`, not `cr` or `calc2`).
- Repeat the procedure or module name after `end` statements (`end module kinds_mod`, `end subroutine compute_residual`) so long units are easy to verify by eye.
- Keep indentation consistent (2 or 4 spaces, matching the project) inside `do`, `if`, `select case`, and module blocks.

## Procedures

- Keep subroutines and functions short and single-purpose — a procedure that computes a residual should not also write output files.
- Use `intent(in)`, `intent(out)`, or `intent(inout)` for every dummy argument; an argument with no `intent` is a red flag that the interface hasn't been thought through.
- Keep functions free of side effects whenever possible (mark them `pure` or `elemental` when they qualify) and reserve `subroutine`s for procedures that mutate state or perform I/O.
- Prefer early validation and clear returns (`if (n <= 0) then; ...; return; end if`) over deeply nested `if` blocks.
- Use `use, only: name1, name2` when importing from a module instead of a bare `use module_name`, so it's clear at the call site exactly what's being pulled in and name collisions are avoided.

## Memory and Arrays

- Prefer allocatable arrays over pointers unless pointer semantics (aliasing, linked structures) are specifically required — allocatables are automatically deallocated and safer by default.
- Check `allocated()` state and array sizes (`size()`, `lbound()`, `ubound()`) before using an array that might not have been allocated yet.
- Deallocate allocatable arrays explicitly when their lifetime isn't naturally scoped (e.g., held in a derived type that outlives a single procedure call); arrays local to a procedure are deallocated automatically on exit.
- Specify array bounds clearly when they matter, especially non-default lower bounds (`real(dp) :: a(0:n)`).
- Avoid unnecessary dynamic allocation inside hot loops — allocate once outside the loop and reuse the buffer, or use automatic/stack arrays for small, fixed-size temporaries.

## Testing and Build

- Use CMake, fpm (the Fortran Package Manager), Make, or whatever build system the project has standardized on — consistently, not a mix.
- Compile with warnings enabled (`gfortran -Wall -Wextra -std=f2008 -fcheck=all` for development builds) and treat important warnings as CI failures.
- Add unit tests for public procedures (a pure function like `standard_deviation` should have a small, fast test with known input/output) and integration tests for full numerical workflows.
- Test boundary conditions (empty arrays, single-element arrays, zero/negative inputs), invalid inputs, and representative scientific cases drawn from the actual problem domain.
- Verify numerical results against a tolerance (e.g., `abs(actual - expected) < 1.0e-10_dp`) rather than relying on exact floating-point equality, which is almost never guaranteed across compilers or optimization levels.

## Common Mistakes

- Declaring variables after executable statements without wrapping them in a `block` construct — Fortran requires all declarations before executable code in a given scoping unit.
- Assuming `random_number` is a function; it is a subroutine (`call random_number(x)`, not `x = random_number()`).
- Writing to stdout (`print`, `write(*,*)`) from a procedure declared `pure` — this is not allowed and will fail to compile.
- Declaring the same variable twice in the same scope, which is easy to miss when a module has grown large.
- Assuming `pi`, `dp`, or other project-standard kind/constant names already exist without importing or defining them explicitly via `use`.
