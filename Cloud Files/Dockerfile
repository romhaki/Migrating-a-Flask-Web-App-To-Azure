# for syntax details, see https://docs.docker.com/engine/reference/builder/
FROM python:3
RUN pip install flask
RUN pip install requests
EXPOSE 5000/tcp
COPY todolist.py .
COPY templates/index.html templates/
CMD python todolist.py