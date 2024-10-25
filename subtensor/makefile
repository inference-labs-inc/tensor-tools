ignition:
# docker run --rm -i quay.io/coreos/butane:release --pretty --strict < butane.yaml > ignition.json
	docker run --rm -v ./:/workspace -w /workspace quay.io/coreos/butane:release --files-dir /workspace --pretty --strict butane.yaml -o ignition.json

clean:
	rm ignition.json
