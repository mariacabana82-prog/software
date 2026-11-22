# software
desarrollar un sistema integral orientado a objetos, sin uso de bases de datos, capaz de gestionar clientes, servicios y reservas para una empresa llamada Software FJ que ofrece varios tipos de servicios (reservas de salas, alquiler de equipos y asesorías especializadas).
from __future__ import annotations

import logging
import re
import uuid
from abc import ABC, abstractmethod
from decimal import Decimal, InvalidOperation, ROUND_HALF_UP
from enum import Enum
from typing import Any, overload


# El archivo de log es el único almacenamiento persistente.
logger = logging.getLogger("software_fj")
logger.setLevel(logging.INFO)

if not logger.handlers:
    handler = logging.FileHandler("software_fj.log", encoding="utf-8")
    handler.setFormatter(logging.Formatter(
        "%(asctime)s | %(levelname)s | %(message)s"
    ))
    logger.addHandler(handler)


# Excepciones personalizadas. Cada error detectado queda registrado.
class ErrorSistema(Exception):
    def __init__(self, mensaje: str):
        logger.error("%s: %s", self.__class__.__name__, mensaje)
        super().__init__(mensaje)


class ErrorValidacion(ErrorSistema):
    pass


class ErrorOperacion(ErrorSistema):
    pass


class ServicioNoDisponible(ErrorOperacion):
    pass


class ErrorCalculo(ErrorSistema):
    pass


class EstadoReservaInvalido(ErrorOperacion):
    pass


def nuevo_id() -> str:
    return str(uuid.uuid4())


def texto_no_vacio(valor: Any, campo: str) -> str:
    if not isinstance(valor, str) or not valor.strip():
        raise ErrorValidacion(f"{campo} debe ser un texto no vacío.")
    return valor.strip()


def decimal_positivo(valor: Any, campo: str) -> Decimal:
    try:
        resultado = Decimal(str(valor))
    except (InvalidOperation, ValueError, TypeError) as exc:
        raise ErrorValidacion(f"{campo} debe ser un número válido.") from exc

    if not resultado.is_finite() or resultado <= 0:
        raise ErrorValidacion(f"{campo} debe ser mayor que cero.")
    return resultado


# Abstracción: entidad general del sistema.
class Entidad(ABC):
    def __init__(self, nombre: str, entidad_id: str | None = None):
        self._id = texto_no_vacio(entidad_id or nuevo_id(), "ID")
        self._nombre = texto_no_vacio(nombre, "Nombre")
        self.validar()

    @property
    def id(self) -> str:
        return self._id

    @property
    def nombre(self) -> str:
        return self._nombre

    @abstractmethod
    def validar(self) -> None:
        """Validación específica de cada tipo de entidad."""


class Cliente(Entidad):
    """Datos personales encapsulados; se exponen mediante propiedades."""

    def __init__(
        self,
        nombre: str,
        email: str,
        telefono: str,
        documento: str,
        entidad_id: str | None = None,
    ):
        self._email = texto_no_vacio(email, "Email").lower()
        self._telefono = texto_no_vacio(telefono, "Teléfono")
        self._documento = texto_no_vacio(documento, "Documento")
        super().__init__(nombre, entidad_id)

    def validar(self) -> None:
        if not re.fullmatch(r"[^@\s]+@[^@\s]+\.[^@\s]+", self._email):
            raise ErrorValidacion("El email no tiene un formato válido.")
        if len(self._telefono) < 7:
            raise ErrorValidacion("El teléfono debe tener al menos 7 caracteres.")
        if len(self._documento) < 4:
            raise ErrorValidacion("El documento debe tener al menos 4 caracteres.")

    @property
    def email(self) -> str:
        return self._email

    @property
    def telefono(self) -> str:
        return self._telefono

    @property
    def documento(self) -> str:
        return self._documento

    def __str__(self) -> str:
        return f"Cliente({self.nombre}, {self.email}, ID={self.id})"


# Servicio es abstracto y define el contrato común de sus derivados.
class Servicio(Entidad, ABC):
    def __init__(
        self,
        nombre: str,
        tarifa_hora: Any,
        max_reservas_activas: int = 1,
        entidad_id: str | None = None,
    ):
        self._tarifa_hora = decimal_positivo(tarifa_hora, "Tarifa por hora")
        if (
            isinstance(max_reservas_activas, bool)
            or not isinstance(max_reservas_activas, int)
            or max_reservas_activas <= 0
        ):
            raise ErrorValidacion("El máximo de reservas debe ser un entero positivo.")

        self._max_reservas_activas = max_reservas_activas
        self._reservas_activas: list[str] = []
        super().__init__(nombre, entidad_id)

    def validar(self) -> None:
        if self._tarifa_hora <= 0:
            raise ErrorValidacion("La tarifa por hora debe ser positiva.")

    @abstractmethod
    def describir(self) -> str:
        pass

    @abstractmethod
    def validar_parametros(self, parametros: dict[str, Any]) -> None:
        pass

    def calcular_costo(
        self,
        duracion_horas: Any,
        impuestos: Any = 0,
        descuento: Any = 0,
    ) -> Decimal:
        """
        Variantes de cálculo mediante parámetros opcionales:
        calcular_costo(horas)
        calcular_costo(horas, impuestos=0.21)
        calcular_costo(horas, impuestos=0.21, descuento=0.10)
        Impuestos y descuentos se expresan como fracción (0.21 = 21%).
        """
        try:
            horas = decimal_positivo(duracion_horas, "Duración")
            impuesto = Decimal(str(impuestos))
            rebaja = Decimal(str(descuento))

            if not impuesto.is_finite() or not Decimal("0") <= impuesto <= Decimal("1"):
                raise ErrorValidacion("Los impuestos deben estar entre 0 y 1.")
            if not rebaja.is_finite() or not Decimal("0") <= rebaja <= Decimal("1"):
                raise ErrorValidacion("El descuento debe estar entre 0 y 1.")

            subtotal = self._tarifa_hora * horas
            total = subtotal * (Decimal("1") - rebaja) * (Decimal("1") + impuesto)
            return total.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
        except ErrorSistema:
            raise
        except Exception as exc:
            # Encadenamiento: el error original queda disponible como causa.
            raise ErrorCalculo("No fue posible calcular el costo del servicio.") from exc

    def registrar_reserva(
        self, reserva_id: str, parametros: dict[str, Any]
    ) -> None:
        self.validar_parametros(parametros)
        if len(self._reservas_activas) >= self._max_reservas_activas:
            raise ServicioNoDisponible(
                f"No hay disponibilidad para el servicio '{self.nombre}'."
            )
        self._reservas_activas.append(reserva_id)

    def liberar_reserva(self, reserva_id: str) -> None:
        if reserva_id in self._reservas_activas:
            self._reservas_activas.remove(reserva_id)

    @property
    def tarifa_hora(self) -> Decimal:
        return self._tarifa_hora


class ServicioSala(Servicio):
    def __init__(self, nombre: str, tarifa_hora: Any, capacidad: int, **kwargs: Any):
        if isinstance(capacidad, bool) or not isinstance(capacidad, int) or capacidad <= 0:
            raise ErrorValidacion("La capacidad de la sala debe ser un entero positivo.")
        self._capacidad = capacidad
        super().__init__(nombre, tarifa_hora, **kwargs)

    def validar_parametros(self, parametros: dict[str, Any]) -> None:
        asistentes = parametros.get("asistentes")
        if isinstance(asistentes, bool) or not isinstance(asistentes, int) or asistentes <= 0:
            raise ErrorValidacion("Indique una cantidad válida de asistentes.")
        if asistentes > self._capacidad:
            raise ErrorValidacion(
                f"La sala admite hasta {self._capacidad} asistentes."
            )

    def describir(self) -> str:
        return f"Sala '{self.nombre}', capacidad {self._capacidad} personas."


class ServicioEquipo(Servicio):
    def __init__(self, nombre: str, tarifa_hora: Any, unidades_disponibles: int, **kwargs: Any):
        if (
            isinstance(unidades_disponibles, bool)
            or not isinstance(unidades_disponibles, int)
            or unidades_disponibles <= 0
        ):
            raise ErrorValidacion("Las unidades disponibles deben ser un entero positivo.")
        self._unidades_disponibles = unidades_disponibles
        self._unidades_totales = unidades_disponibles
        super().__init__(nombre, tarifa_hora, **kwargs)

    def validar_parametros(self, parametros: dict[str, Any]) -> None:
        unidades = parametros.get("unidades")
        if isinstance(unidades, bool) or not isinstance(unidades, int) or unidades <= 0:
            raise ErrorValidacion("Indique una cantidad válida de equipos.")
        if unidades > self._unidades_disponibles:
            raise ServicioNoDisponible(
                f"Solo hay {self._unidades_disponibles} unidades disponibles."
            )

    def registrar_reserva(
        self, reserva_id: str, parametros: dict[str, Any]
    ) -> None:
        self.validar_parametros(parametros)
        if len(self._reservas_activas) >= self._max_reservas_activas:
            raise ServicioNoDisponible(
                f"No hay disponibilidad para el servicio '{self.nombre}'."
            )
        self._unidades_disponibles -= parametros["unidades"]
        self._reservas_activas.append(reserva_id)

    def liberar_reserva(
        self, reserva_id: str, parametros: dict[str, Any] | None = None
    ) -> None:
        if reserva_id in self._reservas_activas:
            self._reservas_activas.remove(reserva_id)
            if parametros:
                self._unidades_disponibles += parametros["unidades"]
                self._unidades_disponibles = min(
                    self._unidades_disponibles, self._unidades_totales
                )

    def describir(self) -> str:
        return (
            f"Alquiler de '{self.nombre}', "
            f"{self._unidades_disponibles}/{self._unidades_totales} unidades disponibles."
        )


class ServicioAsesoria(Servicio):
    MODALIDADES = {"remota", "presencial"}

    def __init__(self, nombre: str, tarifa_hora: Any, especialidad: str, **kwargs: Any):
        self._especialidad = texto_no_vacio(especialidad, "Especialidad")
        super().__init__(nombre, tarifa_hora, **kwargs)

    def validar_parametros(self, parametros: dict[str, Any]) -> None:
        modalidad = parametros.get("modalidad", "remota")
        if modalidad not in self.MODALIDADES:
            raise ErrorValidacion(
                "La modalidad debe ser 'remota' o 'presencial'."
            )

    def describir(self) -> str:
        return f"Asesoría especializada en {self._especialidad}."


class EstadoReserva(Enum):
    PENDIENTE = "pendiente"
    CONFIRMADA = "confirmada"
    CANCELADA = "cancelada"
    PROCESADA = "procesada"


class Reserva:
    def __init__(
        self,
        cliente: Cliente,
        servicio: Servicio,
        duracion_horas: Any,
        parametros: dict[str, Any] | None = None,
        reserva_id: str | None = None,
    ):
        if not isinstance(cliente, Cliente):
            raise ErrorValidacion("La reserva requiere un cliente válido.")
        if not isinstance(servicio, Servicio):
            raise ErrorValidacion("La reserva requiere un servicio válido.")

        self._id = texto_no_vacio(reserva_id or nuevo_id(), "ID de reserva")
        self._cliente = cliente
        self._servicio = servicio
        self._duracion_horas = decimal_positivo(duracion_horas, "Duración")
        self._parametros = dict(parametros or {})
        self._estado = EstadoReserva.PENDIENTE
        self._total: Decimal | None = None

    @property
    def id(self) -> str:
        return self._id

    @property
    def estado(self) -> EstadoReserva:
        return self._estado

    @property
    def total(self) -> Decimal | None:
        return self._total

    def confirmar(
        self, impuestos: Any = 0, descuento: Any = 0
    ) -> Decimal:
        try:
            if self._estado is not EstadoReserva.PENDIENTE:
                raise EstadoReservaInvalido(
                    "Solo se pueden confirmar reservas pendientes."
                )

            self._servicio.validar_parametros(self._parametros)
            total = self._servicio.calcular_costo(
                self._duracion_horas, impuestos, descuento
            )
            self._servicio.registrar_reserva(self.id, self._parametros)
            self._total = total
            self._estado = EstadoReserva.CONFIRMADA
            logger.info("Reserva %s confirmada por %s.", self.id, self._cliente.nombre)
            return total
        except ErrorSistema:
            raise
        except Exception as exc:
            raise ErrorOperacion("Error inesperado al confirmar la reserva.") from exc

    def cancelar(self) -> None:
        if self._estado not in {EstadoReserva.PENDIENTE, EstadoReserva.CONFIRMADA}:
            raise EstadoReservaInvalido(
                "No se puede cancelar una reserva procesada o ya cancelada."
            )

        if self._estado is EstadoReserva.CONFIRMADA:
            self._servicio.liberar_reserva(self.id, self._parametros)

        self._estado = EstadoReserva.CANCELADA
        logger.info("Reserva %s cancelada.", self.id)

    def procesar(self) -> Decimal:
        if self._estado is not EstadoReserva.CONFIRMADA or self._total is None:
            raise EstadoReservaInvalido(
                "Solo se pueden procesar reservas confirmadas."
            )

        total = self._total
        self._servicio.liberar_reserva(self.id, self._parametros)
        self._estado = EstadoReserva.PROCESADA
        logger.info("Reserva %s procesada.", self.id)
        return total

    def __str__(self) -> str:
        return (
            f"Reserva({self.id}, cliente={self._cliente.nombre}, "
            f"servicio={self._servicio.nombre}, estado={self.estado.value})"
        )


class SistemaSoftwareFJ:
    """Gestiona las colecciones en memoria; no utiliza una base de datos."""

    def __init__(self):
        self._clientes: dict[str, Cliente] = {}
        self._servicios: dict[str, Servicio] = {}
        self._reservas: dict[str, Reserva] = {}

    def agregar_cliente(self, cliente: Cliente) -> None:
        if not isinstance(cliente, Cliente):
            raise ErrorValidacion("El objeto proporcionado no es un cliente.")
        if cliente.id in self._clientes:
            raise ErrorOperacion(f"El cliente {cliente.id} ya está registrado.")
        self._clientes[cliente.id] = cliente
        logger.info("Cliente registrado: %s.", cliente.id)

    def agregar_servicio(self, servicio: Servicio) -> None:
        if not isinstance(servicio, Servicio):
            raise ErrorValidacion("El objeto proporcionado no es un servicio.")
        if servicio.id in self._servicios:
            raise ErrorOperacion(f"El servicio {servicio.id} ya está registrado.")
        self._servicios[servicio.id] = servicio
        logger.info("Servicio registrado: %s.", servicio.id)

    def crear_reserva(
        self,
        cliente_id: str,
        servicio_id: str,
        duracion_horas: Any,
        parametros: dict[str, Any] | None = None,
    ) -> Reserva:
        try:
            cliente = self._clientes.get(cliente_id)
            servicio = self._servicios.get(servicio_id)
            if cliente is None:
                raise ErrorValidacion(f"Cliente no encontrado: {cliente_id}.")
            if servicio is None:
                raise ErrorValidacion(f"Servicio no encontrado: {servicio_id}.")

            reserva = Reserva(cliente, servicio, duracion_horas, parametros)
            self._reservas[reserva.id] = reserva
        except ErrorSistema:
            raise
        except Exception as exc:
            raise ErrorOperacion("No fue posible crear la reserva.") from exc
        else:
            logger.info("Reserva creada: %s.", reserva.id)
            return reserva

    def procesar_reserva(self, reserva_id: str) -> Decimal:
        """Demuestra try/except/else/finally y mantiene el sistema operativo."""
        try:
            reserva = self._reservas.get(reserva_id)
            if reserva is None:
                raise ErrorValidacion(f"Reserva no encontrada: {reserva_id}.")
            total = reserva.procesar()
        except ErrorSistema:
            logger.exception("No se pudo procesar la reserva %s.", reserva_id)
            raise
        except Exception as exc:
            raise ErrorOperacion("Fallo inesperado al procesar la reserva.") from exc
        else:
            logger.info("Procesamiento exitoso de la reserva %s.", reserva_id)
            return total
        finally:
            logger.info("Finalizó el intento de procesamiento de %s.", reserva_id)

    def confirmar_reserva(
        self,
        reserva_id: str,
        impuestos: Any = 0,
        descuento: Any = 0,
    ) -> Decimal:
        reserva = self._reservas.get(reserva_id)
        if reserva is None:
            raise ErrorValidacion(f"Reserva no encontrada: {reserva_id}.")
        return reserva.confirmar(impuestos, descuento)

    def cancelar_reserva(self, reserva_id: str) -> None:
        reserva = self._reservas.get(reserva_id)
        if reserva is None:
            raise ErrorValidacion(f"Reserva no encontrada: {reserva_id}.")
        reserva.cancelar()

    def listar_reservas(self) -> list[Reserva]:
        return list(self._reservas.values())

    def listar_servicios(self) -> list[Servicio]:
        return list(self._servicios.values())


if __name__ == "__main__":
    sistema = SistemaSoftwareFJ()

    cliente = Cliente(
        nombre="María Pérez",
        email="maria@example.com",
        telefono="5551234567",
        documento="DOC-1001",
    )
    sala = ServicioSala(
        nombre="Sala de reuniones",
        tarifa_hora="25.00",
        capacidad=12,
    )
    sistema.agregar_cliente(cliente)
    sistema.agregar_servicio(sala)

    reserva = sistema.crear_reserva(
        cliente_id=cliente.id,
        servicio_id=sala.id,
        duracion_horas=2,
        parametros={"asistentes": 8},
    )

    try:
        total = sistema.confirmar_reserva(
            reserva.id,
            impuestos="0.21",
            descuento="0.10",
        )
        print(f"Reserva confirmada. Total: ${total}")
        print(f"Reserva procesada. Total: ${sistema.procesar_reserva(reserva.id)}")
    except ErrorSistema as error:
        # El error ya está registrado; la aplicación puede seguir ejecutándose.
        print(f"No se pudo completar la operación: {error}")
