# rust kube talk at rust london user group 24th of July

https://www.meetup.com/Rust-London-User-Group/events/262999277/

## Usage

```sh
npm i -g reveal-md
reveal-md source/slides.md
```

or just browse the github pages.

## Building

```sh
reveal-md source/slides.md --static docs --static-dirs=source --assets-dir=source --absolute-url https://clux.github.io/rust-meetup-london/
```

Ensure all urls are relative (i.e. start with ./)
