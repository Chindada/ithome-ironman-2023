# ITHOME IRONMAN 2023

[![LICENSE](https://img.shields.io/github/license/Chindada/ithome-ironman-2023?style=for-the-badge)](COPYING)

## Index

### [DAY-01](./day-01/DAY-01.md)

### [DAY-02](./day-02/DAY-02.md)

### [DAY-03](./day-03/DAY-03.md)

### [DAY-04](./day-04/DAY-04.md)

### [DAY-05](./day-05/DAY-05.md)

### [DAY-06](./day-06/DAY-06.md)

### [DAY-07](./day-07/DAY-07.md)

### [DAY-08](./day-08/DAY-08.md)

### [DAY-09](./day-09/DAY-09.md)

### [DAY-10](./day-10/DAY-10.md)

### [DAY-11](./day-11/DAY-11.md)

### [DAY-12](./day-12/DAY-12.md)

### [DAY-13](./day-13/DAY-13.md)

### [DAY-14](./day-14/DAY-14.md)

### [DAY-15](./day-15/DAY-15.md)

### [DAY-16](./day-16/DAY-16.md)

### [DAY-17](./day-17/DAY-17.md)

### [DAY-18](./day-18/DAY-18.md)

### [DAY-19](./day-19/DAY-19.md)

### [DAY-20](./day-20/DAY-20.md)

### [DAY-21](./day-21/DAY-21.md)

### [DAY-22](./day-22/DAY-22.md)

### [DAY-23](./day-23/DAY-23.md)

### [DAY-24](./day-24/DAY-24.md)

### [DAY-25](./day-25/DAY-25.md)

### [DAY-26](./day-26/DAY-26.md)

### [DAY-27](./day-27/DAY-27.md)

### [DAY-28](./day-28/DAY-28.md)

### [DAY-29](./day-29/DAY-29.md)

### [DAY-30](./day-30/DAY-30.md)

## Create

```sh
for i in {01..30}
do
    mkdir "day-${i}"
    touch day-${i}/DAY-${i}.md
    mkdir "day-${i}/assets"
    touch "day-${i}/assets/.gitkeep"
    echo "# DAY ${i}" > day-${i}/DAY-${i}.md
done
```

```sh
for i in {01..30}
do
    echo "### [DAY-${i}](./day-${i}/DAY-${i}.md)"
done
```

## Authors

- [**Tim Hsu**](https://github.com/Chindada)
